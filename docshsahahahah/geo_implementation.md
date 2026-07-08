# DuckDB Geo 功能实现详解

DuckDB 的 Geo（空间/地理）功能是 **直接内建在核心** 中的，`GEOMETRY` 类型是 `LogicalTypeId::GEOMETRY`（值为 60）的一等公民，和 INT、VARCHAR 等基本类型并列注册在类型系统中。

---

## 一、整体架构

```
SQL 输入 'POINT(0 1)'::GEOMETRY
    ↓
[CAST 系统] - geometry.cpp 将 WKT 解析为内部 WKB 格式
    ↓
[类型系统] - LogicalType::GEOMETRY() + GeoTypeInfo (CRS 信息)
    ↓
[存储层] - GeoColumnData 管理持久化，支持三种存储模式
    ↓
[统计/过滤] - GeometryStats 提供 zonemap 剪枝 (type set, extent)
    ↓
[函数系统] - ST_* 函数注册为标量函数
```

### 核心目录结构

| 目录/文件 | 作用 |
|-----------|------|
| `src/common/types/geometry.hpp/.cpp` | 核心类型：Geometry 类、WKT/WKB 解析、shredding |
| `src/common/types/geometry_crs.hpp/.cpp` | 坐标参考系（CRS）支持 |
| `src/function/scalar/geometry/` | ST_* 标量函数实现 |
| `src/function/cast/geo_casts.cpp` | 类型转换逻辑 |
| `src/storage/table/geo_column_data.hpp/.cpp` | 自定义列存储 |
| `src/storage/statistics/geometry_stats.hpp/.cpp` | 统计和 zonemap |
| `extension/parquet/parquet_geometry.cpp` | GeoParquet 支持 |

---

## 二、类型系统

### 几何类型层级

在 `geometry.hpp` 中定义了以下枚举：

```cpp
enum class GeometryType : uint8_t {
    INVALID, POINT, LINESTRING, POLYGON,
    MULTIPOINT, MULTILINESTRING, MULTIPOLYGON,
    GEOMETRYCOLLECTION
};

enum class VertexType : uint8_t {
    XY, XYZ, XYM, XYZM
};
```

坐标顶点分为 `VertexXY`、`VertexXYZ`、`VertexXYM`、`VertexXYZM` 四种结构体，每个都是基于 `double` 的简单容器，编译时就知道维度宽度和 Z/M 标志。

### 物理存储

几何数据 **内部存储为 WKB（Well-Known Binary）格式**，本质上是变长 blob。当用户输入 WKT 字符串时，会立即转换为 WKB 存储。

### CRS（坐标参考系）

`GeoTypeInfo`（继承 `ExtraTypeInfo`）挂载在 `LogicalType` 上：

```cpp
// 创建带 CRS 的 geometry 类型
LogicalType::GEOMETRY("EPSG:4326");
```

支持四种 CRS 格式：`SRID`、`AUTH_CODE`（如 `"EPSG:4326"`）、`PROJJSON`、`WKT2_2019`。

---

## 三、三种存储模式

`GeoColumnData` 是自定义的 `ColumnData` 子类，支持三种布局：

### 1. WKB 模式（默认）
直接将 WKB blob 存到 `StandardColumnData` 中，简单直接。

### 2. Legacy Spatial 模式
兼容旧版 `duckdb_spatial` 扩展的格式，用于从 v1.5 之前的数据库读取数据。

### 3. Shredded 模式（碎化模式）
**这是 DuckDB 的亮点设计**：将几何体拆解为 DuckDB 原生类型：

| 几何类型 | 碎化后的结构 |
|----------|------------|
| `POINT XY` | `STRUCT(x DOUBLE, y DOUBLE)` |
| `LINESTRING XY` | `LIST(STRUCT(x DOUBLE, y DOUBLE))` |
| `POLYGON XY` | `LIST(LIST(STRUCT(x DOUBLE, y DOUBLE)))` |

碎化发生在 **checkpoint 时**，前提条件：
- 满足最小碎化阈值（由 `GeometryMinimumShreddingSize` 设置控制）
- 所有行使用同一几何类型
- 不存在空几何体
- 类型不是 GEOMETRYCOLLECTION

碎化的好处：更好的压缩率、向量化处理、更高效的统计信息。

---

## 四、标量函数（ST_*）

内置了 **7 个空间函数**，定义在 `src/function/scalar/geometry/functions.json` 中，通过代码生成：

| 函数 | 说明 |
|------|------|
| `ST_GeomFromWKB` / `ST_GeomFromText` | 从 WKB/WKT 创建几何体 |
| `ST_AsWKB` / `ST_AsBinary` | 返回 WKB 表示 |
| `ST_AsText` / `ST_AsWKT` | 返回 WKT 文本 |
| `ST_Intersects_Extent` / `&&` | 边界框相交测试 |
| `ST_CRS` | 获取几何体的 CRS 标识 |
| `ST_SetCRS` | 设置几何体的 CRS |
| `vertex_extract` | 从 point 几何体中提取 X/Y/Z/M 坐标 |

每个函数提供了 `DataChunk` 级别的向量化执行函数、`GetFunction()` 工厂方法，以及可选的统计回调（如 `FromWKBStats` 用于推优化）。

### 函数注册方式（声明式）

```json
// functions.json 中定义 -> 代码生成 geometry_functions.hpp
{
    "name": "st_geomfromwkb",
    "parameters": "BLOB",
    "return": "GEOMETRY",
    "function": "GeometryFunction::GeomFromWKB"
}
```

---

## 五、统计与过滤下推

`GeometryStats` 提供了专门的空间统计：

```cpp
struct GeometryStatsData {
    GeometryTypeSet type_set;   // 8种类型 × 4种顶点 = 32位位图
    GeometryExtent extent;      // 边界框 (x/y/z/m min/max)
    GeometryStatsFlags flags;   // 空/非空标记
};
```

优化器可以利用这些统计信息做 **zonemap 剪枝**。例如，当查询 `WHERE ST_Intersects_Extent(geom, 'BOX(0 0, 10 10)')` 时，如果某数据块的 extent 完全不相交，可以直接跳过。

---

## 六、Parquet 集成

`extension/parquet/parquet_geometry.cpp` 实现了 **GeoParquet 元数据读取**，支持从 GeoParquet 文件中读取 GeoArrow 编码的空间数据，解析后输出为碎化后的 STRUCT/LIST 格式。

---

## 七、使用示例

```sql
-- 创建表
CREATE TABLE places (name VARCHAR, geom GEOMETRY);

-- 插入点数据
INSERT INTO places VALUES ('Home', 'POINT(0 1)');

-- 带 CRS 的类型
CREATE TABLE geo_data (g GEOMETRY('EPSG:4326'));

-- 查看 WKT
SELECT name, ST_AsText(geom) FROM places;

-- 提取坐标
SELECT vertex_extract(geom, 'x') FROM places;

-- 边界框过滤
SELECT COUNT(*) FROM places WHERE geom && 'BOX(0 0, 10 10)'::GEOMETRY;
```

---

## 八、关键设计特点总结

1. **原生支持，非扩展**：GEOMETRY 是 DuckDB 核心类型系统的一等公民，不是通过扩展机制添加的
2. **WKB 作为内部格式**：存储紧凑、解析高效、标准兼容
3. **Shredding 碎化**：突破性地将空间数据拆解为 DuckDB 原生类型，充分利用列式存储和向量化执行的优势
4. **CRS 支持**：类型级别携带坐标参考系信息，支持 EPSG 编码等标准
5. **统计驱动优化**：`GeometryTypeSet` + `GeometryExtent` 实现高效的 zonemap 剪枝
6. **GeoParquet 兼容**：紧跟行业标准，支持现代空间数据交换格式

---

## 九、关键文件索引

| 文件 | 说明 |
|------|------|
| `src/include/duckdb/common/types.hpp` | `LogicalTypeId::GEOMETRY`、`LogicalType::GEOMETRY()`、`GeoType` |
| `src/include/duckdb/common/types/geometry.hpp` | `Geometry`、`GeometryType`、`VertexType`、`GeometryExtent` |
| `src/common/types/geometry.cpp` | WKT/WKB 解析、shredding、legacy 转换 (~2428 行) |
| `src/include/duckdb/common/types/geometry_crs.hpp` | `CoordinateReferenceSystem` |
| `src/common/types/geometry_crs.cpp` | CRS 解析与转换 |
| `src/include/duckdb/common/extra_type_info.hpp` | `GeoTypeInfo` |
| `src/function/scalar/geometry/geometry_functions.cpp` | 全部 7 个标量函数实现 |
| `src/function/scalar/geometry/functions.json` | 函数声明式定义 |
| `src/function/cast/geo_casts.cpp` | Geometry 类型转换 |
| `src/storage/table/geo_column_data.cpp` | `GeoColumnData` 存储实现 |
| `src/storage/statistics/geometry_stats.cpp` | 统计信息实现 |
| `extension/parquet/parquet_geometry.cpp` | GeoParquet 读取 |
| `test/sql/types/geo/geometry.test` | 核心类型测试 |
| `test/sql/types/geo/geometry_crs.test` | CRS 测试 |
| `test/geoparquet/geoarrow.test` | GeoParquet 测试 |
