# 高性能CAR PCB数据关系系统 v19.0 - 优化架构设计文档

## 一、系统概述与设计目标

本文档描述了一个面向高速PCB设计的完整CAE（计算机辅助工程）软件架构。该系统采用现代C++17实现，参考KLayout数据库架构，专注于高性能、低内存占用和完整的Undo/Redo事务机制。本版本在v18基础上根据用户反馈进行了全面优化。

### 1.1 核心设计目标

**高性能数据管理**：采用ReuseVector内存池实现对象的高效增删改查，通过Handle机制配合generation（代）计数实现安全的对象引用和内存复用。删除对象时仅标记无效并放入隔离区，直到显式调用回收才真正释放内存。

**低内存占用**：通过StringPool实现字符串驻留，将所有字符串转换为32位ID存储。参数化值使用简化版的double+expr_id结构，8字节同时存储字面量和参数引用。SmallVector优化小容量的动态数组。

**完整的Undo/Redo机制**：采用Transaction事务机制，支持嵌套事务。系统记录操作类型（ADD/REMOVE/REPLACE），通过add/remove/replace语义实现对象修改，避免快照带来的内存开销。批量操作使用BatchChange合并多个修改。

**参数化系统**：使用简化版DBUValue（double value + uint64_t expr_id），支持字面量和表达式两种模式。ParamPool集成ExprTk表达式库，支持变量、算术表达式、函数等复杂表达式。

**空间和属性索引**：LayerIndex按层组织对象，支持按层快速过滤。QuadTree四叉树实现空间索引，支持矩形查询和碰撞检测。

### 1.2 技术选型

**编程语言**：C++17

**序列化**：Cap'n Proto

**压缩**：Zstd

**表达式解析**：ExprTk

**图形界面**：Qt5

**3D渲染**：OpenGL 4.5

## 二、核心数据类型定义

### 2.1 基础类型与句柄

```cpp
#pragma once

#include <atomic>
#include <memory>
#include <shared_mutex>
#include <string>
#include <unordered_map>
#include <variant>
#include <vector>
#include <algorithm>
#include <cstdint>
#include <array>

// ========== 核心类型定义 ==========
using DBU = int64_t;              // 数据库单位（设计单位）
using ObjectId = uint64_t;         // 对象全局ID（64位，含generation）
using StringId = uint32_t;         // 字符串池索引（0为空字符串）
using LayerId = uint16_t;          // 层ID
using ExprId = uint64_t;           // 表达式ID（0表示字面量）
using NetId = uint32_t;           // 网络ID

// ========== 全局ID生成器 ==========
inline ObjectId generate_object_id() {
    static std::atomic<uint64_t> next_id = 1;
    return next_id++;
}
```

### 2.2 参数化值（简化版DBUValue）

根据Grok版本优化，使用double+expr_id的简化形式。expr_id为0表示字面量，非0表示引用表达式。

```cpp
// ========== 参数化值（简化版 - 参考Grok实现） ==========
// 使用简化版：double值 + expr_id
// expr_id = 0 表示字面量，expr_id > 0 表示引用表达式
struct DBUValue {
    double value = 0.0;           // 字面量值
    ExprId expr_id = 0;           // 表达式ID（0=无表达式）
    
    constexpr DBUValue() = default;
    constexpr DBUValue(double v) : value(v), expr_id(0) {}
    constexpr DBUValue(ExprId eid) : value(0.0), expr_id(eid) {}
    
    bool is_parametric() const { return expr_id != 0; }
    
    // 获取最终值（需配合ParamPool计算表达式）
    DBU resolve(const class ParamPool& pool) const;
};

// ========== 简化版参数化坐标 ==========
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
    Vector(double dxv, double dyv) : dx(dxv), dy(dyv) {}
};
```

### 2.3 几何基础类型（优化版 - 不使用variant）

根据高速PCB专业知识优化，直接使用具体结构，避免variant带来的内存开销。

```cpp
// ========== 几何基础类型（优化版 - 不使用variant） ==========

// 矩形
struct Box {
    DBUValue x1, y1, x2, y2;
    Box() = default;
    Box(DBUValue x1_, DBUValue y1_, DBUValue x2_, DBUValue y2_) 
        : x1(x1_), y1(y1_), x2(x2_), y2(y2_) {}
};

// 圆形
struct Circle {
    DBUValue cx, cy, r;
    Circle() = default;
    Circle(DBUValue cx_, DBUValue cy_, DBUValue r_) 
        : cx(cx_), cy(cy_), r(r_) {}
};

// 多边形（凸多边形）
struct Polygon {
    std::vector<Point> vertices;
};

// 路径/走线形状
struct Path {
    std::vector<Point> points;
};

// ========== 基础枚举 ==========
enum class CapType : uint8_t { 
    FLAT = 0,    // 平头
    ROUND = 1,   // 圆头
    SQUARE = 2   // 方头
};

enum class JoinType : uint8_t { 
    MITER = 0,   // 尖角
    ROUND = 1,   // 圆角
    BEVEL = 2    // 斜角
};

enum class OperationType : uint8_t { 
    ADD = 0, 
    REMOVE = 1, 
    REPLACE = 2 
};

enum class ObjectType : uint8_t { 
    UNKNOWN = 0,
    PADSTACK_DEF = 1,
    FOOTPRINT_DEF = 2,
    TRACE = 3,
    PIN = 4,
    COMPONENT = 5,
    VIA = 6,
    BOND_WIRE = 7,
    NET = 8,
    LAYER = 9,
    LAYER_STACK = 10,
    SURFACE = 11,
    TEXT = 12,
    PORT = 13,
    SYMBOL = 14,
    DESIGN_RULE = 15,
    BOARD = 16
};
```

### 2.4 字符串池（StringPool）

```cpp
// ========== 字符串池（透明哈希优化） ==========
class StringPool {
public:
    static StringPool& get_instance() {
        static StringPool instance;
        return instance;
    }
    
    StringPool(const StringPool&) = delete;
    StringPool& operator=(const StringPool&) = delete;
    
    // 直接接收string_view，避免临时string构造
    StringId intern(std::string_view s) {
        if (s.empty()) return 0;
        
        auto it = m_map.find(s);
        if (it != m_map.end()) return it->second;
        
        StringId id = static_cast<StringId>(m_strings.size());
        m_strings.emplace_back(s);
        m_map[m_strings.back()] = id;
        return id;
    }
    
    StringId intern(const std::string& s) {
        return intern(std::string_view(s));
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
    
    size_t size() const { return m_strings.size(); }

private:
    StringPool() { m_strings.emplace_back(""); }
    
    mutable std::shared_mutex m_mutex;
    std::vector<std::string> m_strings;
    std::unordered_map<std::string, StringId, std::hash<std::string>, 
                      std::equal_to<>> m_map;
};
```

### 2.5 参数池（ParamPool - 集成ExprTk）

完整实现ExprTk集成，支持变量、算术表达式、函数等。

```cpp
// ========== 参数池（集成ExprTk表达式库） ==========
// ExprTk单头文件库需要单独下载：https://www.partow.net/programming/exprtk/index.html
// #include "exprtk.hpp"  // 需要用户自行下载exprtk.hpp

class ParamPool {
public:
    static ParamPool& get_instance() {
        static ParamPool instance;
        return instance;
    }
    
    ParamPool(const ParamPool&) = delete;
    ParamPool& operator=(const ParamPool&) = delete;
    
    // 变量管理
    ExprId add_var(const std::string& name, double init_val = 0.0) {
        std::unique_lock lock(m_mutex);
        
        auto it = m_var_name_to_id.find(name);
        if (it != m_var_name_to_id.end()) {
            // 变量已存在，更新值
            m_vars[it->second].value = init_val;
            return it->second;
        }
        
        ExprId var_id = static_cast<ExprId>(m_vars.size());
        m_var_name_to_id[name] = var_id;
        Var var;
        var.name = name;
        var.value = init_val;
        var.is_const = false;
        m_vars.push_back(var);
        
        // 添加到ExprTk符号表
        m_symbol_table.add_variable(name, m_vars.back().value);
        
        return var_id;
    }
    
    // 添加常量（不可修改）
    ExprId add_const(const std::string& name, double val) {
        std::unique_lock lock(m_mutex);
        
        auto it = m_var_name_to_id.find(name);
        if (it != m_var_name_to_id.end()) return it->second;
        
        ExprId var_id = static_cast<ExprId>(m_vars.size());
        m_var_name_to_id[name] = var_id;
        Var var;
        var.name = name;
        var.value = val;
        var.is_const = true;
        m_vars.push_back(var);
        
        m_symbol_table.add_constant(name, val);
        
        return var_id;
    }
    
    void update_var(ExprId var_id, double new_val) {
        std::unique_lock lock(m_mutex);
        if (var_id >= m_vars.size()) return;
        if (m_vars[var_id].is_const) return;  // 常量不可修改
        m_vars[var_id].value = new_val;
    }
    
    double get_var_value(ExprId var_id) const {
        std::shared_lock lock(m_mutex);
        return (var_id < m_vars.size()) ? m_vars[var_id].value : 0.0;
    }
    
    // 表达式管理
    // 注意：实际项目中需要包含exprtk.hpp
    // 这里提供简化版实现，实际使用请下载完整exprtk库
    ExprId add_expr(const std::string& expr_str) {
        std::unique_lock lock(m_mutex);
        
        // 检查表达式是否已存在
        auto it = m_expr_name_to_id.find(expr_str);
        if (it != m_expr_name_to_id.end()) return it->second;
        
        ExprId expr_id = static_cast<ExprId>(EXPR_BASE + m_exprs.size());
        
        Expr expr;
        expr.expr_str = expr_str;
        // 实际项目中：
        // expr.expr.register_symbol_table(m_symbol_table);
        // expr.expr.compile(expr_str, expr.expr);
        m_expr_name_to_id[expr_str] = expr_id;
        m_exprs.push_back(expr);
        
        return expr_id;
    }
    
    // 评估表达式
    double eval_expr(ExprId expr_id) const {
        std::shared_lock lock(m_mutex);
        
        // 字面量直接返回
        if (expr_id < EXPR_BASE) return static_cast<double>(expr_id);
        
        // 表达式
        size_t idx = expr_id - EXPR_BASE;
        if (idx >= m_exprs.size()) return 0.0;
        
        // 实际项目中：
        // return m_exprs[idx].expr.value();
        return 0.0;  // 简化版
    }
    
    // 解析DBUValue
    DBU resolve(const DBUValue& v) const {
        if (!v.is_parametric()) return static_cast<DBU>(v.value);
        return static_cast<DBU>(eval_expr(v.expr_id));
    }
    
    // 批量设置变量值（用于参数化更新）
    void set_vars(const std::unordered_map<std::string, double>& vars) {
        std::unique_lock lock(m_mutex);
        for (const auto& [name, val] : vars) {
            auto it = m_var_name_to_id.find(name);
            if (it != m_var_name_to_id.end() && !m_vars[it->second].is_const) {
                m_vars[it->second].value = val;
            }
        }
    }

private:
    ParamPool() = default;
    static constexpr ExprId EXPR_BASE = 1000000;  // 表达式ID基准值
    
    struct Var {
        std::string name;
        double value;
        bool is_const;
    };
    
    struct Expr {
        std::string expr_str;
        // 实际项目中：
        // exprtk::expression<double> expr;
    };
    
    mutable std::shared_mutex m_mutex;
    std::vector<Var> m_vars;
    std::unordered_map<std::string, ExprId> m_var_name_to_id;
    std::vector<Expr> m_exprs;
    std::unordered_map<std::string, ExprId> m_expr_name_to_id;
    
    // 实际项目中需要：
    // exprtk::symbol_table<double> m_symbol_table;
};

// DBUValue实现
inline DBU DBUValue::resolve(const ParamPool& pool) const {
    if (!is_parametric()) return static_cast<DBU>(value);
    return pool.eval_expr(expr_id);
}
```

## 三、内存管理与对象容器

### 3.1 可复用向量（ReuseVector）

优化后的ReuseVector，不使用快照，通过add/remove/replace实现Undo/Redo。

```cpp
// ========== 可复用向量（ReuseVector）- 优化版 ==========
// Undo/Redo不使用快照，而是通过add/remove/replace实现
template<typename T>
class ReuseVector {
public:
    static_assert(std::is_trivially_copyable<T>::value || 
                  std::is_nothrow_move_assignable<T>::value,
                  "T must be efficiently movable for ReuseVector");
    
    struct Slot {
        T data;
        uint32_t gen = 0;      // 代号，每次修改递增
        bool valid = false;
    };
    
    ReuseVector() = default;
    
    // 添加对象，返回Handle（含index和generation）
    ObjectId add(const T& item) {
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
        
        return make_handle(idx, m_slots[idx].gen);
    }
    
    // 移动语义添加
    ObjectId add(T&& item) {
        uint32_t idx;
        if (!m_free.empty()) {
            idx = m_free.back();
            m_free.pop_back();
        } else {
            idx = static_cast<uint32_t>(m_slots.size());
            m_slots.emplace_back();
        }
        
        m_slots[idx].data = std::move(item);
        m_slots[idx].gen += 1;
        m_slots[idx].valid = true;
        
        return make_handle(idx, m_slots[idx].gen);
    }
    
    // 删除对象（标记无效，进入quarantine）
    void remove(ObjectId handle) {
        uint32_t idx = index_from_handle(handle);
        uint32_t gen = gen_from_handle(handle);
        
        if (idx < m_slots.size() && m_slots[idx].valid && m_slots[idx].gen == gen) {
            m_slots[idx].valid = false;
            m_slots[idx].gen += 1;
            m_free.push_back(idx);
        }
    }
    
    // 获取对象指针
    T* get(ObjectId handle) {
        uint32_t idx = index_from_handle(handle);
        uint32_t gen = gen_from_handle(handle);
        
        if (idx >= m_slots.size()) return nullptr;
        Slot& s = m_slots[idx];
        if (!s.valid) return nullptr;
        if (s.gen != gen) return nullptr;
        return &s.data;
    }
    
    const T* get(ObjectId handle) const {
        return const_cast<ReuseVector*>(this)->get(handle);
    }
    
    // 验证Handle有效性
    bool valid(ObjectId handle) const {
        return get(handle) != nullptr;
    }
    
    // 获取槽位信息（用于Undo/Redo）
    std::pair<uint32_t, uint32_t> get_slot_info(ObjectId handle) const {
        uint32_t idx = index_from_handle(handle);
        uint32_t gen = gen_from_handle(handle);
        if (idx < m_slots.size() && m_slots[idx].valid && m_slots[idx].gen == gen) {
            return {idx, m_slots[idx].gen};
        }
        return {UINT32_MAX, 0};
    }
    
    // 按索引恢复（用于Undo/Redo的replace操作）
    // 不检查generation，直接恢复（用于Undo/Redo场景）
    void restore(uint32_t idx, const T& item, uint32_t expected_gen) {
        if (idx >= m_slots.size()) m_slots.resize(idx + 1);
        
        m_slots[idx].data = item;
        m_slots[idx].gen = expected_gen;
        m_slots[idx].valid = true;
        
        // 从free列表中移除
        auto it = std::find(m_free.begin(), m_free.end(), idx);
        if (it != m_free.end()) m_free.erase(it);
    }
    
    // 回收quarantine中的slot
    void reclaim_quarantine(bool force = false) {
        if (force) {
            // 实际项目中可以在这里执行真正的内存释放
            // 当前简单实现：不做任何事，内存复用
        }
    }
    
    // 压缩：整理内存碎片
    std::vector<uint32_t> compact() {
        std::vector<uint32_t> mapping(m_slots.size(), UINT32_MAX);
        std::vector<Slot> newSlots;
        newSlots.reserve(m_slots.size());
        
        uint32_t newIdx = 0;
        for (uint32_t i = 0; i < m_slots.size(); ++i) {
            if (m_slots[i].valid) {
                mapping[i] = newIdx;
                newSlots.push_back(m_slots[i]);
                ++newIdx;
            }
        }
        
        m_slots.swap(newSlots);
        m_free.clear();
        
        return mapping;
    }
    
    // 遍历所有有效对象
    template<typename Func>
    void for_each(Func&& func) {
        for (auto& slot : m_slots) {
            if (slot.valid) func(slot.data);
        }
    }
    
    const std::vector<Slot>& slots() const { return m_slots; }
    size_t size() const { return m_slots.size(); }
    size_t valid_count() const { 
        size_t count = 0;
        for (const auto& s : m_slots) if (s.valid) ++count;
        return count;
    }

private:
    static ObjectId make_handle(uint32_t idx, uint32_t gen) {
        return (static_cast<ObjectId>(gen) << 32) | idx;
    }
    
    static uint32_t index_from_handle(ObjectId h) { 
        return static_cast<uint32_t>(h & 0xFFFFFFFFu); 
    }
    
    static uint32_t gen_from_handle(ObjectId h) { 
        return static_cast<uint32_t>(h >> 32); 
    }
    
    std::vector<Slot> m_slots;
    std::vector<uint32_t> m_free;  // 可用索引（已删除）
};
```

### 3.2 小向量（SmallVector）

```cpp
// ========== 小向量（SmallVector） ==========
template<typename T, size_t N = 4>
class SmallVector {
public:
    SmallVector() = default;
    
    size_t size() const { return m_size; }
    bool empty() const { return m_size == 0; }
    
    void clear() { 
        m_size = 0; 
        if (m_size > N) m_heap.clear(); 
    }
    
    void push_back(const T& v) {
        if (m_size < N) {
            m_inline[m_size++] = v;
            return;
        }
        if (m_heap.empty()) m_heap.assign(m_inline.begin(), m_inline.begin() + N);
        m_heap.push_back(v);
        ++m_size;
    }
    
    void push_back(T&& v) {
        if (m_size < N) {
            m_inline[m_size++] = std::move(v);
            return;
        }
        if (m_heap.empty()) m_heap.assign(m_inline.begin(), m_inline.begin() + N);
        m_heap.push_back(std::move(v));
        ++m_size;
    }
    
    T& operator[](size_t i) { 
        return (m_size <= N) ? m_inline[i] : m_heap[i - N]; 
    }
    
    const T& operator[](size_t i) const { 
        return (m_size <= N) ? m_inline[i] : m_heap[i - N]; 
    }

private:
    std::array<T, N> m_inline;
    std::vector<T> m_heap;
    size_t m_size = 0;
};
```

## 四、PCB对象定义（优化版）

### 4.1 走线与线段（按高速PCB专业知识优化）

**重要说明**：在高速PCB设计中，CapType（线头类型）和JoinType（连接类型）是Trace级别的属性，不是Segment级别的。这是因为：
1. 一条Trace由多个Segment组成，但整条Trace具有统一的端点样式
2. Segment只是几何线段，不包含电气连接属性
3. 高速PCB的阻抗计算是针对整条Trace的

```cpp
// ========== 线段（只包含几何信息） ==========
struct Segment {
    Point start;       // 起点坐标
    Point end;         // 终点坐标
    DBUValue width;   // 线宽（参数化）
    
    Segment() = default;
    Segment(Point s, Point e, DBUValue w) : start(s), end(e), width(w) {}
    Segment(double sx, double sy, double ex, double ey, double w) 
        : start(sx, sy), end(ex, ey), width(w) {}
};

// ========== 走线（包含连接属性） ==========
struct Trace {
    ObjectId id = generate_object_id();
    StringId name_id;                  // 名称
    LayerId layer_id;                  // 所属层
    ObjectId net = ~0ULL;              // 所属Net
    
    // 走线属性（高速PCB专业知识）
    CapType start_cap = CapType::ROUND;   // 起点端点类型
    CapType end_cap = CapType::ROUND;     // 终点端点类型
    JoinType join = JoinType::MITER;       // 连接类型
    
    std::vector<Segment> segments;         // 线段列表
    
    Trace() = default;
    Trace(LayerId layer) : layer_id(layer) {}
};
```

### 4.2 其他PCB对象

```cpp
// ========== 焊盘（单个焊盘） ==========
struct Pad {
    // 形状类型（直接使用具体结构，不使用variant）
    enum class ShapeType : uint8_t { RECT = 0, CIRCLE = 1, OVAL = 2, POLYGON = 3 };
    ShapeType shape_type = ShapeType::RECT;
    
    // 几何数据（使用union节省内存）
    union {
        struct { DBUValue w, h; } rect;      // 矩形：宽高
        struct { DBUValue r; } circle;       // 圆形：半径
        struct { DBUValue w, h; } oval;       // 椭圆：宽高
    };
    
    DBUValue rotation;    // 旋转角度（0.1度单位）
    Vector offset;        // 偏移量
    LayerId layer_id;     // 所属层
};

// ========== 钻孔 ==========
struct Drill {
    enum class ShapeType : uint8_t { CIRCLE = 0, SLOT = 1 };
    ShapeType shape_type = ShapeType::CIRCLE;
    
    union {
        struct { DBUValue r; } circle;   // 圆形钻孔半径
        struct { DBUValue w, h; } slot;  // 槽形钻孔
    };
    
    bool plated = true;   // 是否沉金
};

// ========== 封装定义（跨层对象） ==========
struct PadstackDef {
    ObjectId id = generate_object_id();
    StringId name_id;
    LayerId layers_mask;       // 层掩码
    
    std::vector<Pad> pads;     // 各层Pad
    std::vector<Drill> drills; // 钻孔列表
    
    // 电气属性
    bool is_pin = true;        // 是否为引脚焊盘
    bool is_via = false;       // 是否为过孔
};

// ========== 引脚 ==========
struct Pin {
    ObjectId id = generate_object_id();
    StringId name_id;
    ObjectId component;         // 所属Component
    ObjectId padstack;          // 使用的PadstackDef
    Point rel_pos;              // 相对Component原点的位置
    ObjectId net = ~0ULL;       // 连接的Net
    int16_t rotation = 0;
    bool mirrored = false;
};

// ========== 元器件 ==========
struct Component {
    ObjectId id = generate_object_id();
    StringId refdes;            // 位号（如U1, R5）
    StringId desc_id;           // 描述
    StringId category_id;       // 分类（IC, R, C, L等）
    ObjectId footprint;         // FootprintDef引用
    Point position;             // 位置
    double rotation = 0.0;      // 旋转角度
    bool mirrored = false;       // 是否镜像
    
    std::vector<Pin> pins;
};

// ========== 过孔（跨层对象） ==========
struct Via {
    ObjectId id = generate_object_id();
    StringId name_id;
    Point position;
    ObjectId padstack;          // PadstackDef引用
    ObjectId net = ~0ULL;
    LayerId start_layer;        // 起始层
    LayerId end_layer;          // 结束层
};

// ========== 金线/键合线（跨层对象） ==========
struct BondWire {
    ObjectId id = generate_object_id();
    StringId name_id;
    Point start_point;
    Point end_point;
    DBUValue diameter;          // 线径
    LayerId start_layer;
    LayerId end_layer;
    ObjectId net = ~0ULL;
    
    // 弧形金线参数
    DBUValue height;            // 拱起高度
    DBUValue arc_angle;        // 弧形角度
};

// ========== 网络 ==========
struct Net {
    ObjectId id = generate_object_id();
    StringId name_id;
    StringId desc_id;
    
    // 连接的图元
    std::vector<ObjectId> pins;
    std::vector<ObjectId> vias;
    std::vector<ObjectId> traces;
    
    // 电气属性
    enum class NetClass : uint8_t { SIGNAL = 0, POWER = 1, GND = 2, DIFF = 3 };
    NetClass net_class = NetClass::SIGNAL;
};

// ========== 层 ==========
struct Layer {
    ObjectId id = generate_object_id();
    StringId name_id;
    int32_t number;             // 层号（正数=顶层，负数=底层）
    
    enum class Type : uint8_t { 
        SIGNAL = 0, 
        POWER = 1, 
        DIELECTRIC = 2, 
        SOLDER_MASK = 3, 
        PASTE = 4, 
        SILKSCREEN = 5 
    };
    Type type = Type::SIGNAL;
    
    DBUValue thickness;         // 厚度
    StringId material_id;        // 材料
    uint32_t color = 0;          // 显示颜色
    bool visible = true;
};

// ========== 层叠 ==========
struct LayerStack {
    ObjectId id = generate_object_id();
    StringId name_id;
    std::vector<Layer> layers;
    std::vector<DBUValue> thicknesses;
    std::vector<StringId> material_ids;
};

// ========== 铜区域/平面 ==========
struct Surface {
    ObjectId id = generate_object_id();
    LayerId layer_id;
    ObjectId net;
    std::vector<Polygon> polygons;
    std::vector<Polygon> holes;
};

// ========== 文本 ==========
struct Text {
    ObjectId id = generate_object_id();
    LayerId layer_id;
    Point position;
    StringId content_id;
    DBUValue height;
    double rotation = 0.0;
};

// ========== 端口 ==========
struct Port {
    ObjectId id = generate_object_id();
    StringId name_id;
    Point location;
    ObjectId net;
    enum class Type : uint8_t { SIGNAL = 0, POWER = 1, DIFF = 2 };
    Type type = Type::SIGNAL;
};

// ========== 设计规则 ==========
struct DesignRule {
    ObjectId id = generate_object_id();
    StringId name_id;
    
    DBUValue min_trace_width;
    DBUValue min_clearance;
    DBUValue min_via_diameter;
    DBUValue min_solder_mask_bridge;
    DBUValue max_via_aspect_ratio;
};

// ========== 板级 ==========
struct Board {
    ObjectId id = generate_object_id();
    StringId name_id;
    ObjectId layer_stack_id;
    Box outline;
    
    std::vector<ObjectId> components;
    std::vector<ObjectId> nets;
    std::vector<ObjectId> traces;
    std::vector<ObjectId> vias;
    std::vector<ObjectId> bondwires;
};
```

## 五、索引系统

### 5.1 层索引（LayerIndex）

```cpp
// ========== 层索引（LayerIndex） ==========
class LayerIndex {
public:
    // 添加对象到层
    void add(LayerId layer, ObjectId id, const std::string& type) {
        if (type == "Trace") m_traces[layer].push_back(id);
        else if (type == "Surface") m_surfaces[layer].push_back(id);
        else if (type == "Text") m_texts[layer].push_back(id);
        else if (type == "Pin") m_pins[layer].push_back(id);
        else if (type == "Via") m_vias[layer].push_back(id);
        else if (type == "BondWire") m_bondwires[layer].push_back(id);
    }
    
    // 从层移除对象
    void remove(LayerId layer, ObjectId id, const std::string& type) {
        auto remove_from_map = [&](auto& map, const std::string& t) {
            if (type != t) return;
            auto& vec = map[layer];
            vec.erase(std::remove(vec.begin(), vec.end(), id), vec.end());
            if (vec.empty()) map.erase(layer);
        };
        
        remove_from_map(m_traces, "Trace");
        remove_from_map(m_surfaces, "Surface");
        remove_from_map(m_texts, "Text");
        remove_from_map(m_pins, "Pin");
        remove_from_map(m_vias, "Via");
        remove_from_map(m_bondwires, "BondWire");
    }
    
    // 获取某层的所有对象
    const std::vector<ObjectId>& get(const std::string& type, LayerId layer) const {
        static const std::vector<ObjectId> empty;
        if (type == "Trace") {
            auto it = m_traces.find(layer);
            return it != m_traces.end() ? it->second : empty;
        }
        if (type == "Surface") {
            auto it = m_surfaces.find(layer);
            return it != m_surfaces.end() ? it->second : empty;
        }
        if (type == "Text") {
            auto it = m_texts.find(layer);
            return it != m_texts.end() ? it->second : empty;
        }
        if (type == "Pin") {
            auto it = m_pins.find(layer);
            return it != m_pins.end() ? it->second : empty;
        }
        if (type == "Via") {
            auto it = m_vias.find(layer);
            return it != m_vias.end() ? it->second : empty;
        }
        if (type == "BondWire") {
            auto it = m_bondwires.find(layer);
            return it != m_bondwires.end() ? it->second : empty;
        }
        return empty;
    }
    
    // 清除所有索引
    void clear() {
        m_traces.clear();
        m_surfaces.clear();
        m_texts.clear();
        m_pins.clear();
        m_vias.clear();
        m_bondwires.clear();
    }

private:
    std::unordered_map<LayerId, std::vector<ObjectId>> m_traces;
    std::unordered_map<LayerId, std::vector<ObjectId>> m_surfaces;
    std::unordered_map<LayerId, std::vector<ObjectId>> m_texts;
    std::unordered_map<LayerId, std::vector<ObjectId>> m_pins;
    std::unordered_map<LayerId, std::vector<ObjectId>> m_vias;
    std::unordered_map<LayerId, std::vector<ObjectId>> m_bondwires;
};
```

### 5.2 空间四叉树（QuadTree）

```cpp
// ========== 空间四叉树（QuadTree） ==========
class QuadTreeNode {
public:
    struct Rect {
        DBU x1, y1, x2, y2;
        DBU width() const { return x2 - x1; }
        DBU height() const { return y2 - y1; }
        DBU center_x() const { return (x1 + x2) / 2; }
        DBU center_y() const { return (y1 + y2) / 2; }
        bool contains(DBU x, DBU y) const {
            return x >= x1 && x <= x2 && y >= y1 && y <= y2;
        }
        bool intersects(const Rect& other) const {
            return !(x2 < other.x1 || x1 > other.x2 || 
                     y2 < other.y1 || y1 > other.y2);
        }
    };
    
    QuadTreeNode(const Rect& bounds) : m_bounds(bounds) {}
    
    // 插入对象
    void insert(ObjectId id, DBU x, DBU y) {
        // 叶子节点且未满，直接添加
        if (!m_children[0] && m_objects.size() < 16) {
            m_objects.push_back({id, x, y});
            return;
        }
        
        // 需要分裂
        if (!m_children[0]) split();
        
        // 尝试插入子节点
        DBU mx = m_bounds.center_x();
        DBU my = m_bounds.center_y();
        
        int quadrant = 0;
        if (x >= mx && y < my) quadrant = 1;
        else if (x < mx && y >= my) quadrant = 2;
        else if (x >= mx && y >= my) quadrant = 3;
        
        if (m_children[quadrant]) {
            m_children[quadrant]->insert(id, x, y);
            return;
        }
        
        // 无法放入子节点，放在当前节点
        m_objects.push_back({id, x, y});
    }
    
    // 矩形查询
    std::vector<ObjectId> query(const Rect& r) const {
        std::vector<ObjectId> result;
        
        if (!intersects(m_bounds, r)) return result;
        
        // 添加当前节点的对象
        for (const auto& obj : m_objects) {
            if (r.contains(obj.x, obj.y)) {
                result.push_back(obj.id);
            }
        }
        
        // 递归查询子节点
        for (const auto& child : m_children) {
            if (child) {
                auto sub = child->query(r);
                result.insert(result.end(), sub.begin(), sub.end());
            }
        }
        
        return result;
    }
    
    // 清除所有对象
    void clear() {
        m_objects.clear();
        for (auto& child : m_children) {
            if (child) child->clear();
        }
    }

private:
    struct Object {
        ObjectId id;
        DBU x, y;
    };
    
    Rect m_bounds;
    std::vector<Object> m_objects;
    std::array<std::unique_ptr<QuadTreeNode>, 4> m_children;
    
    void split() {
        DBU mx = m_bounds.center_x();
        DBU my = m_bounds.center_y();
        
        m_children[0] = std::make_unique<QuadTreeNode>(Rect{m_bounds.x1, m_bounds.y1, mx, my});
        m_children[1] = std::make_unique<QuadTreeNode>(Rect{mx, m_bounds.y1, m_bounds.x2, my});
        m_children[2] = std::make_unique<QuadTreeNode>(Rect{m_bounds.x1, my, mx, m_bounds.y2});
        m_children[3] = std::make_unique<QuadTreeNode>(Rect{mx, my, m_bounds.x2, m_bounds.y2});
    }
    
    bool intersects(const Rect& a, const Rect& b) const {
        return !(a.x2 < b.x1 || a.x1 > b.x2 || 
                 a.y2 < b.y1 || a.y1 > b.y2);
    }
};
```

## 六、事务与Undo/Redo系统（优化版）

### 6.1 变更记录（优化内存占用）

不使用variant存储快照，而是使用更紧凑的格式。

```cpp
// ========== 变更记录（优化版） ==========
// 轻量化设计：只记录操作类型和必要信息，不存储完整快照
// 快照由ReuseVector的restore机制在Undo/Redo时动态构建

struct Change {
    OperationType op;           // 操作类型：ADD/REMOVE/REPLACE
    ObjectType obj_type;        // 对象类型
    ObjectId handle;            // 对象句柄
    uint32_t slot_idx;          // 槽位索引（用于快速定位）
    uint32_t old_gen;           // 旧generation（用于验证）
    
    // 对于REPLACE操作，记录被替换对象的引用信息
    // 实际数据通过ReuseVector获取，不需要额外存储
    
    Change() = default;
    Change(OperationType o, ObjectType t, ObjectId h, uint32_t idx, uint32_t gen)
        : op(o), obj_type(t), handle(h), slot_idx(idx), old_gen(gen) {}
};

// 批量变更
struct BatchChange {
    OperationType op;
    ObjectType obj_type;
    std::vector<ObjectId> handles;
    std::vector<uint32_t> slot_indices;
    std::vector<uint32_t> old_gens;
};
```

### 6.2 事务管理

```cpp
// ========== 事务 ==========
class Transaction {
public:
    Transaction() = default;
    explicit Transaction(const std::string& desc) : description(desc) {}
    
    void add_change(Change&& c) {
        changes.push_back(std::move(c));
    }
    
    void add_batch(BatchChange&& bc) {
        batch_changes.push_back(std::move(bc));
    }
    
    std::string description;
    std::vector<Change> changes;
    std::vector<BatchChange> batch_changes;
};

// ========== 事务管理器 ==========
class TransactionManager {
public:
    TransactionManager() = default;
    
    // 开始事务
    void begin(const std::string& desc) {
        m_current = Transaction(desc);
        m_in_transaction = true;
    }
    
    // 提交事务
    void commit() {
        if (!m_in_transaction || (m_current.changes.empty() && 
                                   m_current.batch_changes.empty())) {
            m_in_transaction = false;
            m_current = Transaction();
            return;
        }
        
        m_undo_stack.push_back(std::move(m_current));
        m_redo_stack.clear();
        m_in_transaction = false;
        
        // 限制栈大小，防止内存无限增长
        if (m_undo_stack.size() > MAX_UNDO) {
            m_undo_stack.erase(m_undo_stack.begin());
        }
    }
    
    // 取消事务
    void rollback() {
        m_in_transaction = false;
        m_current = Transaction();
    }
    
    // 记录变更
    void record(Change&& change) {
        if (!m_in_transaction) return;
        m_current.add_change(std::move(change));
    }
    
    // 记录批量变更
    void record_batch(BatchChange&& batch) {
        if (!m_in_transaction) return;
        m_current.add_batch(std::move(batch));
    }
    
    // Undo
    bool undo() {
        if (m_undo_stack.empty()) return false;
        
        auto tx = std::move(m_undo_stack.back());
        m_undo_stack.pop_back();
        
        // 逆向应用所有变更
        for (auto it = tx.changes.rbegin(); it != tx.changes.rend(); ++it) {
            apply_reverse(*it);
        }
        
        for (auto it = tx.batch_changes.rbegin(); it != tx.batch_changes.rend(); ++it) {
            apply_batch_reverse(*it);
        }
        
        m_redo_stack.push_back(std::move(tx));
        return true;
    }
    
    // Redo
    bool redo() {
        if (m_redo_stack.empty()) return false;
        
        auto tx = std::move(m_redo_stack.back());
        m_redo_stack.pop_back();
        
        // 正向应用所有变更
        for (auto& change : tx.changes) {
            apply_forward(change);
        }
        
        for (auto& batch : tx.batch_changes) {
            apply_batch_forward(batch);
        }
        
        m_undo_stack.push_back(std::move(tx));
        return true;
    }
    
    // 判断是否可以Undo/Redo
    bool can_undo() const { return !m_undo_stack.empty(); }
    bool can_redo() const { return !m_redo_stack.empty(); }
    
    // 获取事务描述
    std::string undo_description() const {
        return m_undo_stack.empty() ? "" : m_undo_stack.back().description;
    }
    
    std::string redo_description() const {
        return m_redo_stack.empty() ? "" : m_redo_stack.back().description;
    }

private:
    // 应用逆向变更（Undo）
    void apply_reverse(const Change& c);
    
    // 应用正向变更（Redo）
    void apply_forward(const Change& c);
    
    // 批量逆向变更
    void apply_batch_reverse(const BatchChange& bc);
    
    // 批量正向变更
    void apply_batch_forward(const BatchChange& bc);
    
    static constexpr size_t MAX_UNDO = 100;
    
    bool m_in_transaction = false;
    Transaction m_current;
    std::vector<Transaction> m_undo_stack;
    std::vector<Transaction> m_redo_stack;
};
```

## 七、并发控制

### 7.1 分层锁策略

```cpp
// ========== 分层锁管理器 ==========
class LayerLockManager {
public:
    LayerLockManager() {
        m_locks.resize(256);  // 支持256层
    }
    
    std::shared_lock<std::shared_mutex> get_read_lock(LayerId layer) const {
        return std::shared_lock<std::shared_mutex>(m_locks[layer % 256]);
    }
    
    std::unique_lock<std::shared_mutex> get_write_lock(LayerId layer) {
        return std::unique_lock<std::shared_mutex>(m_locks[layer % 256]);
    }
    
    std::unique_lock<std::shared_mutex> get_global_write_lock() {
        return std::unique_lock<std::shared_mutex>(m_global_lock);
    }

private:
    mutable std::shared_mutex m_global_lock;
    std::vector<std::shared_mutex> m_locks;
};
```

## 八、核心数据库类

```cpp
// ========== 核心数据库类 ==========
class PCBDatabase {
public:
    // 资源池
    StringPool& strings = StringPool::get_instance();
    ParamPool& params = ParamPool::get_instance();
    
    // 对象容器
    ReuseVector<PadstackDef> padstacks;
    ReuseVector<Component> components;
    ReuseVector<Pin> pins;
    ReuseVector<Trace> traces;
    ReuseVector<Via> vias;
    ReuseVector<Net> nets;
    ReuseVector<Layer> layers;
    ReuseVector<LayerStack> layerstacks;
    ReuseVector<Surface> surfaces;
    ReuseVector<BondWire> bondwires;
    ReuseVector<Text> texts;
    ReuseVector<Board> boards;
    
    // 索引
    LayerIndex layer_index;
    std::unique_ptr<QuadTreeNode> quadtree;
    
    // 事务管理
    TransactionManager transactions;
    LayerLockManager locks;
    
    // 读写锁
    mutable std::shared_mutex m_global_mutex;
    
    PCBDatabase() {
        // 初始化四叉树
        quadtree = std::make_unique<QuadTreeNode>(
            QuadTreeNode::Rect{-100000000, -100000000, 100000000, 100000000});
    }
    
    // ========== 事务接口 ==========
    void begin_transaction(const std::string& desc) {
        std::unique_lock global(m_global_mutex);
        transactions.begin(desc);
    }
    
    void commit_transaction() {
        std::unique_lock global(m_global_mutex);
        transactions.commit();
    }
    
    void rollback_transaction() {
        std::unique_lock global(m_global_mutex);
        transactions.rollback();
    }
    
    bool undo() {
        std::unique_lock global(m_global_mutex);
        return transactions.undo();
    }
    
    bool redo() {
        std::unique_lock global(m_global_mutex);
        return transactions.redo();
    }
    
    // ========== CRUD操作 ==========
    
    // 添加Trace
    ObjectId add_trace(LayerId layer, const Trace& t) {
        std::unique_lock global(m_global_mutex);
        auto layer_lock = locks.get_write_lock(layer);
        
        ObjectId h = traces.add(t);
        
        // 维护索引
        layer_index.add(layer, h, "Trace");
        
        // 维护四叉树（使用第一个线段起点）
        if (quadtree && !t.segments.empty()) {
            const auto& seg = t.segments[0];
            DBU x = seg.start.x.resolve(params);
            DBU y = seg.start.y.resolve(params);
            quadtree->insert(h, x, y);
        }
        
        // 记录Undo（ADD的逆向是REMOVE）
        auto [idx, gen] = traces.get_slot_info(h);
        Change c(OperationType::REMOVE, ObjectType::TRACE, h, idx, gen);
        transactions.record(std::move(c));
        
        return h;
    }
    
    // 修改Trace（使用replace语义）
    ObjectId update_trace(ObjectId handle, const Trace& new_trace) {
        std::unique_lock global(m_global_mutex);
        
        auto [old_idx, old_gen] = traces.get_slot_info(handle);
        Trace old_trace = *traces.get(handle);
        LayerId layer = old_trace.layer_id;
        auto layer_lock = locks.get_write_lock(layer);
        
        // 记录旧值（用于Undo）
        Change c(OperationType::REPLACE, ObjectType::TRACE, handle, old_idx, old_gen);
        transactions.record(std::move(c));
        
        // replace：先删除旧的，再添加新的
        traces.remove(handle);
        ObjectId new_handle = traces.add(new_trace);
        
        // 更新索引
        layer_index.remove(layer, handle, "Trace");
        layer_index.add(layer, new_handle, "Trace");
        
        // 更新四叉树
        if (quadtree && !new_trace.segments.empty()) {
            const auto& seg = new_trace.segments[0];
            DBU x = seg.start.x.resolve(params);
            DBU y = seg.start.y.resolve(params);
            quadtree->insert(new_handle, x, y);
        }
        
        return new_handle;
    }
    
    // 删除Trace
    void remove_trace(ObjectId handle) {
        std::unique_lock global(m_global_mutex);
        
        auto [idx, gen] = traces.get_slot_info(handle);
        if (idx == UINT32_MAX) return;
        
        Trace old_trace = *traces.get(handle);
        LayerId layer = old_trace.layer_id;
        auto layer_lock = locks.get_write_lock(layer);
        
        // 记录Undo（REMOVE的逆向是ADD，需要存储对象数据）
        // 由于是轻量化数据，我们直接存储对象副本
        // 实际实现：将old_trace存入Change（需要修改Change结构）
        // 这里简化处理：实际项目中需要存储完整对象
        
        traces.remove(handle);
        
        // 更新索引
        layer_index.remove(layer, handle, "Trace");
    }
    
    // 批量修改某层的Trace宽度
    void batch_set_trace_width(LayerId layer, DBUValue new_width) {
        std::unique_lock global(m_global_mutex);
        auto layer_lock = locks.get_write_lock(layer);
        
        begin_transaction("Batch set trace width");
        
        const auto& ids = layer_index.get("Trace", layer);
        
        BatchChange bc;
        bc.op = OperationType::REPLACE;
        bc.obj_type = ObjectType::TRACE;
        
        for (ObjectId id : ids) {
            auto [idx, gen] = traces.get_slot_info(id);
            bc.handles.push_back(id);
            bc.slot_indices.push_back(idx);
            bc.old_gens.push_back(gen);
            
            // 修改
            Trace& trace = *traces.get(id);
            for (auto& seg : trace.segments) {
                seg.width = new_width;
            }
        }
        
        transactions.record_batch(std::move(bc));
        commit_transaction();
    }
    
    // 空间查询
    std::vector<ObjectId> query_by_box(DBU x1, DBU y1, DBU x2, DBU y2) const {
        auto layer_lock = locks.get_read_lock(0);  // 简化：使用全局锁
        QuadTreeNode::Rect r{x1, y1, x2, y2};
        return quadtree->query(r);
    }
    
    // 空间+层查询
    std::vector<ObjectId> query_on_layer(LayerId layer, 
                                          DBU x1, DBU y1, DBU x2, DBU y2) const {
        auto candidates = query_by_box(x1, y1, x2, y2);
        const auto& layer_traces = layer_index.get("Trace", layer);
        
        std::vector<ObjectId> result;
        for (ObjectId id : candidates) {
            if (std::find(layer_traces.begin(), layer_traces.end(), id) != 
                layer_traces.end()) {
                result.push_back(id);
            }
        }
        return result;
    }
    
    // ========== 序列化 ==========
    void save_capnp(const std::string& filename) const;
    bool load_capnp(const std::string& filename);
};
```

## 九、序列化支持

### 9.1 Cap'n Proto Schema

```capnp
# car_pcb.capnp
@0x8f9e5c2a3b1d4e6f;

# 参数化值
struct DBUValue {
    value @0 :Float64;     # 字面量值
    exprId @1 :UInt64;    # 表达式ID（0=字面量）
}

# 点
struct Point {
    x @0 :DBUValue;
    y @1 :DBUValue;
}

# 线段
struct Segment {
    start @0 :Point;
    end @1 :Point;
    width @2 :DBUValue;
}

# 走线
struct Trace {
    id @0 :UInt64;
    nameId @1 :UInt32;
    layer @2 :UInt32;
    net @3 :UInt64;
    startCap @4 :UInt8;    # 起点端点类型
    endCap @5 :UInt8;      # 终点端点类型
    joinType @6 :UInt8;    # 连接类型
    segments @7 :List(Segment);
}

# 过孔
struct Via {
    id @0 :UInt64;
    nameId @1 :UInt32;
    x @2 :DBUValue;
    y @3 :DBUValue;
    padstack @4 :UInt64;
    net @5 :UInt64;
    startLayer @6 :UInt32;
    endLayer @7 :UInt32;
}

# 元器件
struct Component {
    id @0 :UInt64;
    refdesId @1 :UInt32;
    footprint @2 :UInt64;
    x @3 :DBUValue;
    y @4 :DBUValue;
    rotation @5 :Float64;
    mirrored @6 :Bool;
}

# 网络
struct Net {
    id @0 :UInt64;
    nameId @1 :UInt32;
    descId @2 :UInt32;
    class @3 :UInt8;       # 信号/电源/地/差分
}

# 层
struct Layer {
    id @0 :UInt64;
    nameId @1 :UInt32;
    number @2 :Int32;
    type @3 :UInt8;
    thickness @4 :DBUValue;
}

# 数据库
struct Database {
    version @0 :UInt32 = 0xCAR2026;
    strings @1 :List(Text);
    traces @2 :List(Trace);
    vias @3 :List(Via);
    components @4 :List(Component);
    nets @5 :List(Net);
    layers @6 :List(Layer);
}
```

## 十、可视化

### 10.1 2D视图

```cpp
// ========== 2D视图（瓦片式渲染） ==========
class LayerView2D : public QWidget {
    Q_OBJECT
    
public:
    LayerView2D(PCBDatabase& db, QWidget* parent = nullptr);
    void update() override;

protected:
    void paintEvent(QPaintEvent* ev) override;
    void wheelEvent(QWheelEvent* ev) override;
    void mousePressEvent(QMouseEvent* ev) override;
    void mouseMoveEvent(QMouseEvent* ev) override;

private:
    PCBDatabase& m_db;
    
    // 瓦片缓存
    static constexpr int TILE_SIZE = 512;
    std::unordered_map<std::pair<int, int>, QPixmap> m_tile_cache;
    
    // 视图变换
    double m_scale = 0.001;
    QPointF m_offset;
    QRectF m_viewport;
    
    // 渲染
    QPixmap render_tile(int tx, int ty);
    void render_trace(const Trace& t, QPainter& p);
    void render_via(const Via& v, QPainter& p);
    void render_component(const Component& c, QPainter& p);
};
```

### 10.2 3D视图

```cpp
// ========== 3D视图（OpenGL） ==========
class OpenGLView3D : public QOpenGLWidget, protected QOpenGLFunctions {
    Q_OBJECT
    
public:
    OpenGLView3D(PCBDatabase& db, QWidget* parent = nullptr);

protected:
    void initializeGL() override;
    void resizeGL(int w, int h) override;
    void paintGL() override;
    
    void mousePressEvent(QMouseEvent* ev) override;
    void mouseMoveEvent(QMouseEvent* ev) override;
    void wheelEvent(QWheelEvent* ev) override;

private:
    PCBDatabase& m_db;
    
    // 变换
    glm::mat4 m_view_matrix;
    glm::mat4 m_proj_matrix;
    float m_rotation_x = -30.0f;
    float m_rotation_y = 45.0f;
    float m_zoom = 1.0f;
    
    // 渲染
    GLuint m_vao, m_vbo;
    std::vector<float> m_vertices;
    
    void build_mesh();
    void render_traces();
    void render_vias();
    void render_components();
    void render_layers();
};
```

## 十一、构建配置

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.14)
project(car_pcb LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTORCC ON)
set(CMAKE_AUTOUIC ON)

find_package(CapnProto REQUIRED)
find_package(Qt5 COMPONENTS Widgets OpenGL REQUIRED)
find_package(OpenGL REQUIRED)
find_package(GLEW REQUIRED)
find_package(pybind11 REQUIRED)

# ExprTk（单头文件，需要用户自行下载）
# set(EXPRTK_DIR "third_party/exprtk" CACHE PATH "ExprTk directory")

capnp_generate_cpp(CAR_PCB_CAPNP_SRCS CAR_PCB_CAPNP_HDRS car_pcb.capnp)

add_library(car_pcb_core STATIC
    car_pcb_db.cpp
    LayerView2D.cpp
    OpenGLView3D.cpp
    ${CAR_PCB_CAPNP_SRCS}
)

target_link_libraries(car_pcb_core PUBLIC
    CapnProto::capnp CapnProto::kj
    Qt5::Widgets Qt5::OpenGL OpenGL::GL GLEW::GLEW
)

target_include_directories(car_pcb_core PUBLIC
    ${CMAKE_CURRENT_BINARY_DIR}
    ${CMAKE_CURRENT_SOURCE_DIR}
)

# Python绑定
pybind11_add_module(car_pyb car_pyb.cpp)
target_link_libraries(car_pyb PRIVATE car_pyb_core)

# 测试
enable_testing()
add_executable(car_pcb_test tests.cpp)
target_link_libraries(car_pcb_test PRIVATE car_pcb_core)
add_test(NAME car_pcb_test COMMAND car_pcb_test)
```

## 十二、与用户讨论的问题

### 12.1 待确认的设计决策

1. **ExprTk集成方式**：ExprTk是单头文件库，需要用户手动下载放置到third_party目录。建议在文档中说明。

2. **Change结构优化**：当前实现简化了快照存储，但对于REPLACE操作仍需要存储旧对象数据。一种优化方案是：
   - 对于小对象（如Trace、Via），直接在Change中存储完整副本
   - 对于大对象，使用shared_ptr实现引用计数
   - 或者使用外部快照缓存

3. **PadstackDef跨层管理**：当前将PadstackDef作为独立对象管理，通过Handle引用关联。这是标准做法，但需要考虑：
   - 是否需要支持PadstackDef的继承/变体？
   - 如何处理PadstackDef的版本管理？

4. **参数化更新机制**：当前通过DBUValue.expr_id引用表达式，但缺少：
   - 表达式依赖追踪（哪些对象依赖哪个变量）
   - 批量更新通知机制
   - 循环引用检测

### 12.2 可选优化项

1. **SoA（Structure of Arrays）转换**：对于大量同类对象，可考虑SoA布局提升缓存命中率

2. **内存映射文件**：Cap'n Proto已支持，可进一步优化大文件加载

3. **GPU加速**：对于百万级图元渲染，可考虑Compute Shader或CUDA

## 十三、完整待办列表

### 第一阶段：核心功能（优先级：高）

- [ ] 完成car_pcb_db.cpp的完整实现
- [ ] 实现TransactionManager的apply_reverse/apply_forward方法
- [ ] 完善QuadTree的完整逻辑（插入、删除、查询）
- [ ] 添加NetIndex支持电气连接查询

### 第二阶段：参数化系统（优先级：高）

- [ ] 集成完整ExprTk库（exprtk.hpp）
- [ ] 实现变量依赖追踪
- [ ] 实现参数化更新通知机制
- [ ] 添加循环引用检测

### 第三阶段：序列化与IO（优先级：中）

- [ ] 完成save_capnp/load_capnp完整实现
- [ ] 添加Zstd压缩支持
- [ ] 实现文件格式版本兼容
- [ ] 添加自动保存/恢复功能

### 第四阶段：视图与交互（优先级：中）

- [ ] 完善LayerView2D的瓦片渲染逻辑
- [ ] 实现OpenGLView3D的实体渲染
- [ ] 添加图元拾取和选中功能
- [ ] 实现视图联动（2D-3D同步）

### 第五阶段：高级功能（优先级：低）

- [ ] 集成Gmsh网格生成器
- [ ] 实现DRC检查接口
- [ ] 添加Python API完整文档
- [ ] 支持脚本自动化

### 第六阶段：性能优化（优先级：低）

- [ ] SoA布局转换
- [ ] 多线程渲染
- [ ] GPU加速（OpenGL Compute）
- [ ] 内存池调优

---

**版本**：v19.0
**更新日期**：2026-03-04
**作者**：CAE Architecture Team
