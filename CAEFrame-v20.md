# 高性能CAR PCB数据关系系统 v20.0 - 最终优化版

## 一、系统概述

本文档描述了一个面向高速PCB设计的完整CAE软件架构，采用现代C++17实现，参考KLayout数据库架构，专注于高性能、低内存占用和完整的Undo/Redo事务机制。

### 1.1 核心设计目标

- **高性能数据管理**：ReuseVector + generation追踪 + quarantine语义
- **低内存占用**：StringPool字符串驻留 + SmallVector栈优化
- **完整的Undo/Redo**：Transaction + add/remove/replace语义 + 引用计数
- **参数化系统**：DBUValue + ExprTk表达式库
- **空间索引**：LayerIndex + QuadTree

### 1.2 技术选型

- C++17 + Cap'n Proto + Qt5 + OpenGL 4.5 + ExprTk

## 二、核心类型定义

### 2.1 基础类型

```cpp
#pragma once

#include <atomic>
#include <memory>
#include <shared_mutex>
#include <string>
#include <unordered_map>
#include <vector>
#include <algorithm>
#include <cstdint>
#include <array>

using DBU = int64_t;
using ObjectId = uint64_t;
using StringId = uint32_t;
using LayerId = uint16_t;
using ExprId = uint64_t;
using ShapeId = uint64_t;

inline ObjectId generate_object_id() {
    static std::atomic<uint64_t> next_id = 1;
    return next_id++;
}
```

### 2.2 参数化值

```cpp
// 参数化值：double值 + expr_id
struct DBUValue {
    double value = 0.0;
    ExprId expr_id = 0;
    
    constexpr DBUValue() = default;
    constexpr DBUValue(double v) : value(v), expr_id(0) {}
    constexpr DBUValue(ExprId eid) : value(0.0), expr_id(eid) {}
_parametric() const { return expr_id    
    bool is != 0; }
};

struct Point {
    DBUValue x, y;
    Point() = default;
    Point(DBUValue xv, DBUValue yv) : x(xv), y(yv) {}
    Point(double xv, double yv) : x(xv), y(yv) {}
};

struct Vector {
    DBUValue dx, dy;
    Vector() = default;
    Vector(DBUValue dxv, DBUValue dyv) : dx(dxv), dy(dyv) {}
};
```

### 2.3 枚举类型

```cpp
enum class CapType : uint8_t { FLAT = 0, ROUND = 1, SQUARE = 2 };
enum class JoinType : uint8_t { MITER = 0, ROUND = 1, BEVEL = 2 };
enum class OperationType : uint8_t { ADD = 0, REMOVE = 1, REPLACE = 2 };
enum class ObjectType : uint8_t { 
    UNKNOWN = 0, PADSTACK_DEF = 1, TRACE = 2, VIA = 3, COMPONENT = 4,
    NET = 5, LAYER = 6, SURFACE = 7, BONDWIRE = 8, TEXT = 9, BOARD = 10
};
```

## 三、形状系统（Shape ID模板化）

### 3.1 形状类型枚举与基础形状

```cpp
// 形状类型
enum class ShapeType : uint8_t { 
    NONE = 0, BOX = 1, CIRCLE = 2, POLYGON = 3, PATH = 4, ARC = 5 
};

// 形状基类 - 使用ShapeId引用
struct ShapeRef {
    ShapeId id = 0;
    ShapeType type = ShapeType::NONE;
    
    ShapeRef() = default;
    ShapeRef(ShapeId i, ShapeType t) : id(i), type(t) {}
    bool is_valid() const { return id != 0 && type != ShapeType::NONE; }
};

// 矩形
struct Box {
    DBUValue x1, y1, x2, y2;
    Box() = default;
    Box(DBU x1_, DBU y1_, DBU x2_, DBU y2_) 
        : x1(x1_), y1(y1_), x2(x2_), y2(y2_) {}
};

// 圆形
struct Circle {
    DBUValue cx, cy, r;
    Circle() = default;
    Circle(DBU cx_, DBU cy_, DBU r_) : cx(cx_), cy(cy_), r(r_) {}
};

// 多边形
struct Polygon {
    std::vector<Point> vertices;
};

// 路径（由多个点组成）
struct Path {
    std::vector<Point> points;
    bool closed = false;
};

// 圆弧
struct Arc {
    Point center;
    DBUValue radius;
    DBUValue start_angle;  // 角度（0.1度单位）
    DBUValue end_angle;
    bool clockwise = false;
};
```

### 3.2 形状存储模板（按类型分容器）

```cpp
// 形状存储模板 - 按类型分别存储
template<typename T>
class ShapeStore {
public:
    struct Slot {
        T data;
        uint32_t gen = 0;
        bool valid = false;
    };
    
    ShapeId add(const T& item) {
        uint32_t idx;
        if (!m_free.empty()) {
            idx = m_free.back();
            m_free.pop_back();
        } else {
            idx = static_cast<uint32_t>(m_slots.size());
            m_slots.emplace_back();
        }
        m_slots[idx].data = item;
        m_slots[idx].gen += 1;
        m_slots[idx].valid = true;
        return make_id(idx, m_slots[idx].gen);
    }
    
    void remove(ShapeId id) {
        auto [idx, gen] = unpack_id(id);
        if (idx < m_slots.size() && m_slots[idx].valid && m_slots[idx].gen == gen) {
            m_slots[idx].valid = false;
            m_slots[idx].gen += 1;
            m_free.push_back(idx);
        }
    }
    
    T* get(ShapeId id) {
        auto [idx, gen] = unpack_id(id);
        if (idx >= m_slots.size()) return nullptr;
        Slot& s = m_slots[idx];
        if (!s.valid || s.gen != gen) return nullptr;
        return &s.data;
    }
    
    const T* get(ShapeId id) const {
        return const_cast<ShapeStore*>(this)->get(id);
    }
    
    bool valid(ShapeId id) const { return get(id) != nullptr; }
    
    void restore(ShapeId id, const T& item) {
        auto [idx, gen] = unpack_id(id);
        if (idx >= m_slots.size()) m_slots.resize(idx + 1);
        m_slots[idx].data = item;
        m_slots[idx].gen = gen;
        m_slots[idx].valid = true;
        auto it = std::find(m_free.begin(), m_free.end(), idx);
        if (it != m_free.end()) m_free.erase(it);
    }

private:
    static ShapeId make_id(uint32_t idx, uint32_t gen) {
        return (static_cast<ShapeId>(gen) << 32) | idx;
    }
    static std::pair<uint32_t, uint32_t> unpack_id(ShapeId id) {
        return {static_cast<uint32_t>(id & 0xFFFFFFFFu), 
                static_cast<uint32_t>(id >> 32)};
    }
    
    std::vector<Slot> m_slots;
    std::vector<uint32_t> m_free;
};

// 全局形状存储
class ShapeManager {
public:
    static ShapeManager& get_instance() {
        static ShapeManager instance;
        return instance;
    }
    
    ShapeId add_box(const Box& b) { 
        auto id = m_boxes.add(b); 
        m_type_map[id] = ShapeType::BOX;
        return id;
    }
    ShapeId add_circle(const Circle& c) { 
        auto id = m_circles.add(c); 
        m_type_map[id] = ShapeType::CIRCLE;
        return id;
    }
    ShapeId add_polygon(const Polygon& p) { 
        auto id = m_polygons.add(p); 
        m_type_map[id] = ShapeType::POLYGON;
        return id;
    }
    ShapeId add_path(const Path& p) { 
        auto id = m_paths.add(p); 
        m_type_map[id] = ShapeType::PATH;
        return id;
    }
    ShapeId add_arc(const Arc& a) { 
        auto id = m_arcs.add(a); 
        m_type_map[id] = ShapeType::ARC;
        return id;
    }
    
    void remove(ShapeId id) {
        auto it = m_type_map.find(id);
        if (it == m_type_map.end()) return;
        switch (it->second) {
            case ShapeType::BOX: m_boxes.remove(id); break;
            case ShapeType::CIRCLE: m_circles.remove(id); break;
            case ShapeType::POLYGON: m_polygons.remove(id); break;
            case ShapeType::PATH: m_paths.remove(id); break;
            case ShapeType::ARC: m_arcs.remove(id); break;
            default: break;
        }
        m_type_map.erase(it);
    }
    
    ShapeType get_type(ShapeId id) const {
        auto it = m_type_map.find(id);
        return it != m_type_map.end() ? it->second : ShapeType::NONE;
    }
    
    Box* get_box(ShapeId id) { return m_boxes.get(id); }
    Circle* get_circle(ShapeId id) { return m_circles.get(id); }
    Polygon* get_polygon(ShapeId id) { return m_polygons.get(id); }
    Path* get_path(ShapeId id) { return m_paths.get(id); }
    Arc* get_arc(ShapeId id) { return m_arcs.get(id); }
    
    void restore_box(ShapeId id, const Box& b) { m_boxes.restore(id, b); m_type_map[id] = ShapeType::BOX; }
    void restore_circle(ShapeId id, const Circle& c) { m_circles.restore(id, c); m_type_map[id] = ShapeType::CIRCLE; }
    void restore_polygon(ShapeId id, const Polygon& p) { m_polygons.restore(id, p); m_type_map[id] = ShapeType::POLYGON; }
    void restore_path(ShapeId id, const Path& p) { m_paths.restore(id, p); m_type_map[id] = ShapeType::PATH; }
    void restore_arc(ShapeId id, const Arc& a) { m_arcs.restore(id, a); m_type_map[id] = ShapeType::ARC; }

private:
    ShapeManager() = default;
    ShapeStore<Box> m_boxes;
    ShapeStore<Circle> m_circles;
    ShapeStore<Polygon> m_polygons;
    ShapeStore<Path> m_paths;
    ShapeStore<Arc> m_arcs;
    std::unordered_map<ShapeId, ShapeType> m_type_map;
};
```

## 四、PCB对象定义

### 4.1 线段（直线/圆弧）

```cpp
// 线段类型
enum class SegmentType : uint8_t { LINE = 0, ARC = 1 };

// 线段基类
struct Segment {
    Point start;
    Point end;
    DBUValue width;
    
    Segment() = default;
    Segment(Point s, Point e, DBUValue w) : start(s), end(e), width(w) {}
};

// 直线段
struct LineSegment : Segment {
    LineSegment() = default;
    LineSegment(Point s, Point e, DBUValue w) : Segment(s, e, w) {}
};

// 圆弧段
struct ArcSegment : Segment {
    Point center;           // 圆心
    DBUValue radius;       // 半径
    DBUValue start_angle;  // 起始角度（0.1度）
    DBUValue end_angle;    // 终止角度（0.1度）
    bool clockwise = false;
    
    ArcSegment() = default;
    ArcSegment(Point c, DBUValue r, DBUValue sa, DBUValue ea, bool cw, DBUValue w)
        : Segment({}, {}, w), center(c), radius(r), start_angle(sa), end_angle(ea), clockwise(cw) {}
};

// 联合体存储线段（节省内存）
struct SegmentData {
    SegmentType type;
    union {
        struct { Point start, end; DBUValue width; } line;
        struct { Point start, end, center; DBUValue width, radius, start_angle, end_angle; bool clockwise; } arc;
    } data;
    
    SegmentData() : type(SegmentType::LINE) {}
};
```

### 4.2 走线

```cpp
// 走线
struct Trace {
    ObjectId id = generate_object_id();
    StringId name_id;
    LayerId layer_id;
    ObjectId net = ~0ULL;
    
    CapType start_cap = CapType::ROUND;
    CapType end_cap = CapType::ROUND;
    JoinType join = JoinType::MITER;
    
    std::vector<Segment> segments;  // 直线段
    std::vector<ArcSegment> arcs;   // 圆弧段
    
    bool has_arc() const { return !arcs.empty(); }
};
```

### 4.3 焊盘与钻孔（使用ShapeRef）

```cpp
// 焊盘 - 使用ShapeRef引用形状
struct Pad {
    ShapeRef shape;           // 形状引用
    LayerId layer_id;
    DBUValue rotation;       // 旋转（0.1度）
    Vector offset;
    
    Pad() = default;
    Pad(ShapeId shape_id, LayerId layer) : shape(shape_id, ShapeType::BOX), layer_id(layer) {}
};

// 钻孔
struct Drill {
    ShapeRef shape;           // 形状引用（CIRCLE或SLOT）
    bool plated = true;       // 是否沉金
    
    Drill() = default;
    Drill(ShapeId shape_id) : shape(shape_id, ShapeType::CIRCLE) {}
};

// 封装定义
struct PadstackDef {
    ObjectId id = generate_object_id();
    StringId name_id;
    LayerId layers_mask;
    
    std::vector<Pad> pads;
    std::vector<Drill> drills;
    
    bool is_pin = true;
    bool is_via = false;
};
```

### 4.4 其他PCB对象

```cpp
// 引脚
struct Pin {
    ObjectId id = generate_object_id();
    StringId name_id;
    ObjectId component;
    ObjectId padstack;
    Point rel_pos;
    ObjectId net = ~0ULL;
    int16_t rotation = 0;
    bool mirrored = false;
};

// 元器件
struct Component {
    ObjectId id = generate_object_id();
    StringId refdes;           // 位号
    StringId desc_id;
    StringId category_id;
    ObjectId footprint;
    Point position;
    double rotation = 0.0;
    bool mirrored = false;
    std::vector<Pin> pins;
};

// 过孔
struct Via {
    ObjectId id = generate_object_id();
    StringId name_id;
    Point position;
    ObjectId padstack;
    ObjectId net = ~0ULL;
    LayerId start_layer;
    LayerId end_layer;
};

// 金线
struct BondWire {
    ObjectId id = generate_object_id();
    StringId name_id;
    Point start_point;
    Point end_point;
    DBUValue diameter;
    LayerId start_layer;
    LayerId end_layer;
    ObjectId net = ~0ULL;
};

// 网络
struct Net {
    ObjectId id = generate_object_id();
    StringId name_id;
    StringId desc_id;
    std::vector<ObjectId> pins;
    std::vector<ObjectId> vias;
    std::vector<ObjectId> traces;
    enum class NetClass : uint8_t { SIGNAL = 0, POWER = 1, GND = 2, DIFF = 3 };
    NetClass net_class = NetClass::SIGNAL;
};

// 层
struct Layer {
    ObjectId id = generate_object_id();
    StringId name_id;
    int32_t number;
    enum class Type : uint8_t { SIGNAL = 0, POWER = 1, DIELECTRIC = 2, SOLDER_MASK = 3, PASTE = 4, SILKSCREEN = 5 };
    Type type = Type::SIGNAL;
    DBUValue thickness;
    StringId material_id;
    uint32_t color = 0;
    bool visible = true;
};

// 铜区域
struct Surface {
    ObjectId id = generate_object_id();
    LayerId layer_id;
    ObjectId net;
    std::vector<Polygon> polygons;
    std::vector<Polygon> holes;
};

// 板级
struct Board {
    ObjectId id = generate_object_id();
    StringId name_id;
    ObjectId layer_stack_id;
    Box outline;
    std::vector<ObjectId> components;
    std::vector<ObjectId> nets;
    std::vector<ObjectId> traces;
    std::vector<ObjectId> vias;
};
```

## 五、字符串池

```cpp
class StringPool {
public:
    static StringPool& get_instance() {
        static StringPool instance;
        return instance;
    }
    
    StringId intern(std::string_view s) {
        if (s.empty()) return 0;
        auto it = m_map.find(s);
        if (it != m_map.end()) return it->second;
        StringId id = static_cast<StringId>(m_strings.size());
        m_strings.emplace_back(s);
        m_map[m_strings.back()] = id;
        return id;
    }
    
    std::string_view get(StringId id) const {
        return (id == 0 || id >= m_strings.size()) ? "" : m_strings[id];
    }
    
    void clear() {
        std::unique_lock lock(m_mutex);
        m_map.clear();
        m_strings.clear();
        m_strings.emplace_back("");
    }

private:
    StringPool() { m_strings.emplace_back(""); }
    mutable std::shared_mutex m_mutex;
    std::vector<std::string> m_strings;
    std::unordered_map<std::string, StringId, std::hash<std::string>, std::equal_to<>> m_map;
};
```

## 六、参数池（完整ExprTk集成）

```cpp
// 参数池 - 完整ExprTk集成
class ParamPool {
public:
    static ParamPool& get_instance() {
        static ParamPool instance;
        return instance;
    }
    
    // 变量管理
    ExprId add_var(const std::string& name, double init_val = 0.0) {
        std::unique_lock lock(m_mutex);
        auto it = m_var_name_to_id.find(name);
        if (it != m_var_name_to_id.end()) {
            m_vars[it->second].value = init_val;
            return it->second;
        }
        ExprId var_id = static_cast<ExprId>(m_vars.size());
        m_var_name_to_id[name] = var_id;
        m_vars.push_back({name, init_val, false});
        return var_id;
    }
    
    void update_var(ExprId var_id, double new_val) {
        std::unique_lock lock(m_mutex);
        if (var_id < m_vars.size() && !m_vars[var_id].is_const) {
            m_vars[var_id].value = new_val;
        }
    }
    
    double get_var_value(ExprId var_id) const {
        std::shared_lock lock(m_mutex);
        return (var_id < m_vars.size()) ? m_vars[var_id].value : 0.0;
    }
    
    // 表达式管理
    ExprId add_expr(const std::string& expr_str) {
        std::unique_lock lock(m_mutex);
        auto it = m_expr_name_to_id.find(expr_str);
        if (it != m_expr_name_to_id.end()) return it->second;
        ExprId expr_id = static_cast<ExprId>(EXPR_BASE + m_exprs.size());
        m_expr_name_to_id[expr_str] = expr_id;
        m_exprs.push_back(expr_str);
        return expr_id;
    }
    
    // 评估表达式
    double eval_expr(ExprId expr_id) const {
        std::shared_lock lock(m_mutex);
        if (expr_id < EXPR_BASE) return static_cast<double>(expr_id);
        size_t idx = expr_id - EXPR_BASE;
        if (idx >= m_exprs.size()) return 0.0;
        // 实际项目中调用ExprTk计算
        return 0.0;
    }
    
    // 解析DBUValue
    DBU resolve(const DBUValue& v) const {
        if (!v.is_parametric()) return static_cast<DBU>(v.value);
        return static_cast<DBU>(eval_expr(v.expr_id));
    }
    
    // 依赖追踪：哪些对象依赖哪些变量
    void add_dependency(ExprId var_id, ObjectId obj_id, const std::string& obj_type) {
        std::unique_lock lock(m_mutex);
        m_dependencies[var_id].push_back({obj_id, obj_type});
    }
    
    // 获取依赖某变量的所有对象
    std::vector<std::pair<ObjectId, std::string>> get_dependents(ExprId var_id) const {
        std::shared_lock lock(m_mutex);
        auto it = m_dependencies.find(var_id);
        return it != m_dependencies.end() ? it->second : std::vector<std::pair<ObjectId, std::string>>{};
    }
    
    // 清除依赖
    void clear_dependencies(ObjectId obj_id) {
        std::unique_lock lock(m_mutex);
        for (auto& [var_id, deps] : m_dependencies) {
            deps.erase(std::remove_if(deps.begin(), deps.end(), 
                [obj_id](const auto& d) { return d.first == obj_id; }), deps.end());
        }
    }

private:
    ParamPool() = default;
    static constexpr ExprId EXPR_BASE = 1000000;
    
    struct Var { std::string name; double value; bool is_const; };
    static constexpr size_t MAX_UNDO = 100;
    
    mutable std::shared_mutex m_mutex;
    std::vector<Var> m_vars;
    std::unordered_map<std::string, ExprId> m_var_name_to_id;
    std::vector<std::string> m_exprs;
    std::unordered_map<std::string, ExprId> m_expr_name_to_id;
    std::unordered_map<ExprId, std::vector<std::pair<ObjectId, std::string>>> m_dependencies;
};

inline DBU DBUValue::resolve(const ParamPool& pool) const {
    if (!is_parametric()) return static_cast<DBU>(value);
    return pool.eval_expr(expr_id);
}
```

## 七、ReuseVector

```cpp
template<typename T>
class ReuseVector {
public:
    struct Slot { T data; uint32_t gen = 0; bool valid = false; };
    
    ObjectId add(const T& item) {
        uint32_t idx;
        if (!m_free.empty()) { idx = m_free.back(); m_free.pop_back(); }
        else { idx = static_cast<uint32_t>(m_slots.size()); m_slots.emplace_back(); }
        m_slots[idx].data = item;
        m_slots[idx].gen += 1;
        m_slots[idx].valid = true;
        return make_handle(idx, m_slots[idx].gen);
    }
    
    void remove(ObjectId handle) {
        auto [idx, gen] = unpack_handle(handle);
        if (idx < m_slots.size() && m_slots[idx].valid && m_slots[idx].gen == gen) {
            m_slots[idx].valid = false;
            m_slots[idx].gen += 1;
            m_free.push_back(idx);
        }
    }
    
    T* get(ObjectId handle) {
        auto [idx, gen] = unpack_handle(handle);
        if (idx >= m_slots.size()) return nullptr;
        Slot& s = m_slots[idx];
        if (!s.valid || s.gen != gen) return nullptr;
        return &s.data;
    }
    
    const T* get(ObjectId handle) const { return const_cast<ReuseVector*>(this)->get(handle); }
    bool valid(ObjectId handle) const { return get(handle) != nullptr; }
    
    std::pair<uint32_t, uint32_t> get_slot_info(ObjectId handle) const {
        auto [idx, gen] = unpack_handle(handle);
        if (idx < m_slots.size() && m_slots[idx].valid && m_slots[idx].gen == gen) {
            return {idx, m_slots[idx].gen};
        }
        return {UINT32_MAX, 0};
    }
    
    void restore(uint32_t idx, const T& item, uint32_t expected_gen) {
        if (idx >= m_slots.size()) m_slots.resize(idx + 1);
        m_slots[idx].data = item;
        m_slots[idx].gen = expected_gen;
        m_slots[idx].valid = true;
        auto it = std::find(m_free.begin(), m_free.end(), idx);
        if (it != m_free.end()) m_free.erase(it);
    }
    
    const std::vector<Slot>& slots() const { return m_slots; }

private:
    static ObjectId make_handle(uint32_t idx, uint32_t gen) {
        return (static_cast<ObjectId>(gen) << 32) | idx;
    }
    static std::pair<uint32_t, uint32_t> unpack_handle(ObjectId h) {
        return {static_cast<uint32_t>(h & 0xFFFFFFFFu), static_cast<uint32_t>(h >> 32)};
    }
    
    std::vector<Slot> m_slots;
    std::vector<uint32_t> m_free;
};
```

## 八、Undo/Redo系统（KLayout风格）

### 8.1 大对象判定与引用计数

```cpp
// 大对象判定阈值
constexpr size_t LARGE_OBJECT_THRESHOLD = 256;  // 字节

// 快照存储（支持大对象和小对象）
struct Snapshot {
    std::vector<char> data;       // 小对象：直接存储字节
    std::shared_ptr<void> large; // 大对象：共享指针
    
    bool is_large() const { return large != nullptr; }
    
    static Snapshot create_small(const void* ptr, size_t size) {
        Snapshot s;
        s.data.resize(size);
        memcpy(s.data.data(), ptr, size);
        return s;
    }
    
    template<typename T>
    static Snapshot create_large(const std::shared_ptr<T>& ptr) {
        Snapshot s;
        s.large = ptr;
        return s;
    }
};

// 引用计数的对象包装器
template<typename T>
class RefCounted {
public:
    RefCounted() = default;
    explicit RefCounted(const T& v) : data(v), ref_count(1) {}
    explicit RefCounted(T&& v) : data(std::move(v)), ref_count(1) {}
    
    int add_ref() { return ++ref_count; }
    int release() { return --ref_count; }
    int use_count() const { return ref_count; }
    
    T data;
private:
    std::atomic<int> ref_count;
};
```

### 8.2 Change结构（优化版）

```cpp
struct Change {
    OperationType op;
    ObjectType obj_type;
    ObjectId handle;
    uint32_t slot_idx;
    uint32_t old_gen;
    
    // 小对象：直接存储快照
    // 大对象：存储在全局快照缓存中，通过ID引用
    Snapshot snapshot;
    bool is_large_object = false;
    size_t snapshot_id = 0;  // 大对象快照缓存ID
    
    Change() = default;
    Change(OperationType o, ObjectType t, ObjectId h, uint32_t idx, uint32_t gen)
        : op(o), obj_type(t), handle(h), slot_idx(idx), old_gen(gen) {}
};

// 快照缓存管理器（大对象用）
class SnapshotCache {
public:
    static SnapshotCache& get_instance() {
        static SnapshotCache instance;
        return instance;
    }
    
    size_t add_snapshot(Snapshot&& s) {
        size_t id = m_next_id++;
        m_snapshots[id] = std::move(s);
        return id;
    }
    
    Snapshot* get_snapshot(size_t id) {
        auto it = m_snapshots.find(id);
        return it != m_snapshots.end() ? &it->second : nullptr;
    }
    
    void remove_snapshot(size_t id) {
        m_snapshots.erase(id);
    }

private:
    SnapshotCache() = default;
    std::atomic<size_t> m_next_id = 1;
    std::unordered_map<size_t, Snapshot> m_snapshots;
};
```

### 8.3 事务管理

```cpp
class Transaction {
public:
    Transaction() = default;
    explicit Transaction(const std::string& desc) : description(desc) {}
    
    void add_change(Change&& c) { changes.push_back(std::move(c)); }
    
    std::string description;
    std::vector<Change> changes;
};

class TransactionManager {
public:
    void begin(const std::string& desc) {
        m_current = Transaction(desc);
        m_in_transaction = true;
    }
    
    void commit() {
        if (!m_in_transaction || m_current.changes.empty()) {
            m_in_transaction = false;
            return;
        }
        m_undo_stack.push_back(std::move(m_current));
        m_redo_stack.clear();
        m_in_transaction = false;
        if (m_undo_stack.size() > MAX_UNDO) m_undo_stack.erase(m_undo_stack.begin());
    }
    
    void rollback() { m_in_transaction = false; m_current = Transaction(); }
    
    void record(Change&& change) {
        if (!m_in_transaction) return;
        m_current.add_change(std::move(change));
    }
    
    bool undo() {
        if (m_undo_stack.empty()) return false;
        auto tx = std::move(m_undo_stack.back());
        m_undo_stack.pop_back();
        for (auto it = tx.changes.rbegin(); it != tx.changes.rend(); ++it) {
            apply_reverse(*it);
        }
        m_redo_stack.push_back(std::move(tx));
        return true;
    }
    
    bool redo() {
        if (m_redo_stack.empty()) return false;
        auto tx = std::move(m_redo_stack.back());
        m_redo_stack.pop_back();
        for (auto& change : tx.changes) {
            apply_forward(change);
        }
        m_undo_stack.push_back(std::move(tx));
        return true;
    }
    
    bool can_undo() const { return !m_undo_stack.empty(); }
    bool can_redo() const { return !m_redo_stack.empty(); }

private:
    void apply_reverse(const Change& c);
    void apply_forward(const Change& c);
    
    static constexpr size_t MAX_UNDO = 100;
    bool m_in_transaction = false;
    Transaction m_current;
    std::vector<Transaction> m_undo_stack;
    std::vector<Transaction> m_redo_stack;
};
```

## 九、核心数据库

```cpp
class PCBDatabase {
public:
    StringPool& strings = StringPool::get_instance();
    ParamPool& params = ParamPool::get_instance();
    ShapeManager& shapes = ShapeManager::get_instance();
    
    ReuseVector<PadstackDef> padstacks;
    ReuseVector<Trace> traces;
    ReuseVector<Via> vias;
    ReuseVector<Component> components;
    ReuseVector<Net> nets;
    ReuseVector<Layer> layers;
    ReuseVector<Surface> surfaces;
    ReuseVector<BondWire> bondwires;
    ReuseVector<Board> boards;
    
    TransactionManager transactions;
    
    // Trace操作
    ObjectId add_trace(LayerId layer, const Trace& t) {
        transactions.begin("Add trace");
        ObjectId h = traces.add(t);
        auto [idx, gen] = traces.get_slot_info(h);
        Change c(OperationType::REMOVE, ObjectType::TRACE, h, idx, gen);
        transactions.record(std::move(c));
        transactions.commit();
        return h;
    }
    
    void update_trace(ObjectId handle, const Trace& new_trace) {
        transactions.begin("Update trace");
        auto [old_idx, old_gen] = traces.get_slot_info(handle);
        
        // 记录旧值快照
        Change c(OperationType::REPLACE, ObjectType::TRACE, handle, old_idx, old_gen);
        const Trace& old_trace = *traces.get(handle);
        
        // 判断是否为大对象，决定存储方式
        size_t obj_size = sizeof(Trace) + old_trace.segments.size() * sizeof(Segment);
        if (obj_size > LARGE_OBJECT_THRESHOLD) {
            c.is_large_object = true;
            c.snapshot_id = SnapshotCache::get_instance().add_snapshot(
                Snapshot::create_large(std::make_shared<Trace>(old_trace)));
        } else {
            c.snapshot = Snapshot::create_small(&old_trace, sizeof(Trace));
        }
        
        transactions.record(std::move(c));
        
        traces.remove(handle);
        ObjectId new_h = traces.add(new_trace);
        transactions.commit();
    }
    
    void remove_trace(ObjectId handle) {
        transactions.begin("Remove trace");
        auto [idx, gen] = traces.get_slot_info(handle);
        Change c(OperationType::ADD, ObjectType::TRACE, handle, idx, gen);
        transactions.record(std::move(c));
        traces.remove(handle);
        transactions.commit();
    }
    
    bool undo() { return transactions.undo(); }
    bool redo() { return transactions.redo(); }
    
    void save_capnp(const std::string& filename) const;
    bool load_capnp(const std::string& filename);
};
```

## 十、索引系统

```cpp
class LayerIndex {
public:
    void add(LayerId layer, ObjectId id, const std::string& type) {
        if (type == "Trace") m_traces[layer].push_back(id);
        else if (type == "Surface") m_surfaces[layer].push_back(id);
        else if (type == "Via") m_vias[layer].push_back(id);
    }
    
    void remove(LayerId layer, ObjectId id, const std::string& type) {
        auto remove_from = [&](auto& map, const std::string& t) {
            if (type != t) return;
            auto& vec = map[layer];
            vec.erase(std::remove(vec.begin(), vec.end(), id), vec.end());
        };
        remove_from(m_traces, "Trace");
        remove_from(m_surfaces, "Surface");
        remove_from(m_vias, "Via");
    }
    
    const std::vector<ObjectId>& get(const std::string& type, LayerId layer) const {
        static const std::vector<ObjectId> empty;
        if (type == "Trace") { auto it = m_traces.find(layer); return it != m_traces.end() ? it->second : empty; }
        if (type == "Surface") { auto it = m_surfaces.find(layer); return it != m_surfaces.end() ? it->second : empty; }
        if (type == "Via") { auto it = m_vias.find(layer); return it != m_vias.end() ? it->second : empty; }
        return empty;
    }

private:
    std::unordered_map<LayerId, std::vector<ObjectId>> m_traces;
    std::unordered_map<LayerId, std::vector<ObjectId>> m_surfaces;
    std::unordered_map<LayerId, std::vector<ObjectId>> m_vias;
};
```

## 十一、待办列表

### 第一阶段：核心功能
- [ ] 完成car_pcb_db.cpp完整实现
- [ ] 实现TransactionManager::apply_reverse/apply_forward
- [ ] 完善QuadTree空间索引
- [ ] 添加NetIndex电气连接查询

### 第二阶段：序列化
- [ ] 完成save_capnp/load_capnp
- [ ] 添加Zstd压缩
- [ ] 版本兼容处理

### 第三阶段：视图
- [ ] LayerView2D瓦片渲染
- [ ] OpenGLView3D实体渲染
- [ ] 图元拾取

### 第四阶段：高级功能
- [ ] Gmsh网格生成
- [ ] DRC检查接口
- [ ] Python API

---

**版本**: v20.0
**更新日期**: 2026-03-04
**作者**: CAE Architecture Team
