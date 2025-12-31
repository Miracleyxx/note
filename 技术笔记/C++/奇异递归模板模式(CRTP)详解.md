# 奇异递归模板模式(CRTP)详解

## 1. 基本概念

CRTP(Curiously Recurring Template Pattern，奇异递归模板模式)是一种C++高级模板编程技术，其特点是派生类将自身作为模板参数传递给其基类。

```cpp
template <typename Derived>
class Base {
    // 基类实现...
};

class Derived : public Base<Derived> {
    // 派生类实现...
};
```

## 2. 工作原理

- 基类模板接收派生类类型作为模板参数
- 基类可以通过`static_cast`安全地向下转换`this`指针，访问派生类的方法和属性
- 编译器为每个派生类生成不同的基类模板实例
- 实现编译期(静态)多态，避免运行时多态的开销

```cpp
template <typename Derived>
class Base {
public:
    void interface() {
        // 安全地转换为派生类指针并调用其方法
        static_cast<Derived*>(this)->implementation();
    }
};
```

## 3. 常见应用场景

### 3.1 静态多态（避免虚函数开销）

```cpp
template <typename Derived>
class Shape {
public:
    void draw() {
        static_cast<Derived*>(this)->drawImpl();
    }
};

class Circle : public Shape<Circle> {
public:
    void drawImpl() { 
        std::cout << "画圆" << std::endl;
    }
};

class Rectangle : public Shape<Rectangle> {
public:
    void drawImpl() {
        std::cout << "画矩形" << std::endl;
    }
};
```

### 3.2 对象计数器（线程安全版本）

```cpp
template <typename Derived>
class Counter {
private:
    static std::atomic<size_t> count;
public:
    Counter() { ++count; }
    ~Counter() { --count; }
    static size_t getCount() { return count.load(); }
};

template <typename Derived>
std::atomic<size_t> Counter<Derived>::count(0);

class Widget : public Counter<Widget> {};
class Button : public Counter<Button> {};

// 使用
Widget w1, w2;
Button b1;
std::cout << "Widget数量: " << Widget::getCount() << std::endl; // 输出 2
std::cout << "Button数量: " << Button::getCount() << std::endl; // 输出 1
```

### 3.3 运算符重载复用

```cpp
template <typename Derived>
class Comparable {
public:
    // 只需派生类实现 < 和 ==，其他比较运算符自动提供
    bool operator>(const Derived& other) const {
        const Derived& self = static_cast<const Derived&>(*this);
        return !(self < other) && !(self == other);
    }
    
    bool operator<=(const Derived& other) const {
        const Derived& self = static_cast<const Derived&>(*this);
        return (self < other) || (self == other);
    }
    
    bool operator>=(const Derived& other) const {
        const Derived& self = static_cast<const Derived&>(*this);
        return !(self < other);
    }
    
    bool operator!=(const Derived& other) const {
        const Derived& self = static_cast<const Derived&>(*this);
        return !(self == other);
    }
};

class Point : public Comparable<Point> {
private:
    int x, y;
public:
    Point(int x, int y) : x(x), y(y) {}
    
    bool operator<(const Point& other) const {
        return (x < other.x) || ((x == other.x) && (y < other.y));
    }
    
    bool operator==(const Point& other) const {
        return (x == other.x) && (y == other.y);
    }
};
```

### 3.4 混入(Mixin)技术

```cpp
// 序列化功能混入
template <typename Derived>
class Serializable {
public:
    void serialize() {
        static_cast<Derived*>(this)->serializeImpl();
        // 所有序列化对象共享的通用序列化逻辑
        std::cout << "完成序列化，数据已保存" << std::endl;
    }
};

// 日志功能混入
template <typename Derived>
class Loggable {
public:
    void log(const std::string& message) {
        std::cout << "[LOG] " << message << " 对象: ";
        static_cast<Derived*>(this)->logIdentity();
    }
};

// 同时混入多种功能
class User : 
    public Serializable<User>,
    public Loggable<User> 
{
public:
    std::string name;
    int age;
    
    void serializeImpl() {
        std::cout << "序列化用户: " << name << ", " << age << std::endl;
    }
    
    void logIdentity() {
        std::cout << "用户 " << name << std::endl;
    }
};
```

## 4. CRTP与虚函数对比

### 4.1 实现方式对比

**虚函数方式**:
```cpp
class Shape {
public:
    virtual void draw() = 0; // 纯虚函数
    virtual ~Shape() {}
};

class Circle : public Shape {
public:
    void draw() override {
        std::cout << "画圆" << std::endl;
    }
};
```

**CRTP方式**:
```cpp
template <typename Derived>
class Shape {
public:
    void draw() {
        static_cast<Derived*>(this)->drawImpl();
    }
};

class Circle : public Shape<Circle> {
public:
    void drawImpl() {
        std::cout << "画圆" << std::endl;
    }
};
```

### 4.2 详细对比

| 特性                 | CRTP (静态多态)              | 虚函数 (动态多态)              |
| -------------------- | ---------------------------- | ------------------------------ |
| 多态实现时机         | 编译期                       | 运行时                         |
| 性能开销             | 无额外开销                   | 虚表查找开销                   |
| 内存结构             | 无虚表                       | 每个对象增加虚表指针           |
| 内联可能性           | 高 (编译器易于内联)          | 低 (通常不内联虚函数调用)      |
| 异质集合             | 不支持存储不同类型于同一容器 | 支持通过基类指针操作不同派生类 |
| 灵活性               | 编译期确定，不能动态变更     | 运行时多态，可动态替换         |
| 编码复杂性           | 较高，使用模板和静态转换     | 较低，标准OOP概念              |
| 编译时间与二进制大小 | 可能增加编译时间和代码体积   | 通常不显著增加                 |

## 5. 混入(Mixin)技术详解

混入(Mixin)是一种代码复用技术，允许类"混入"(组合)其他类的功能，而不建立严格的继承关系。CRTP提供了C++中实现混入的强大方式。

### 5.1 混入与传统继承对比

**传统多重继承**:
- 容易导致钻石继承问题
- 可能产生类型歧义
- 所有子类共享同一个基类实例

**CRTP混入**:
- 每个派生类实例化出不同的基类模板
- 避免钻石继承问题
- 更灵活的代码组织方式

### 5.2 多重混入示例

```cpp
// 多个功能混入到单个类中
class Product : 
    public Serializable<Product>,
    public Loggable<Product>,
    public Comparable<Product>,
    public Counter<Product>
{
public:
    int id;
    std::string name;
    double price;
    
    // 实现各个混入所需的接口
    void serializeImpl() { /* ... */ }
    void logIdentity() { /* ... */ }
    bool operator<(const Product& other) const { /* ... */ }
    bool operator==(const Product& other) const { /* ... */ }
};
```

## 6. 高级CRTP技巧

### 6.1 无宏定义的对象计数器

```cpp
template <typename Derived>
class ObjectCounter {
public:
    ObjectCounter() { ++getCounter(); }
    ~ObjectCounter() { --getCounter(); }
    static size_t count() { return getCounter().load(); }

private:
    static std::atomic<size_t>& getCounter() {
        static std::atomic<size_t> instance(0);
        return instance;
    }
};

class Widget : public ObjectCounter<Widget> {};
```

### 6.2 用于策略模式的CRTP

```cpp
// 不同排序策略的基类
template <typename Derived>
class SortPolicy {
public:
    template <typename Container>
    void sort(Container& container) {
        static_cast<Derived*>(this)->sortImpl(container);
    }
};

// 快速排序策略
class QuickSort : public SortPolicy<QuickSort> {
public:
    template <typename Container>
    void sortImpl(Container& container) {
        std::cout << "使用快速排序" << std::endl;
        // 快排实现...
    }
};

// 合并排序策略
class MergeSort : public SortPolicy<MergeSort> {
public:
    template <typename Container>
    void sortImpl(Container& container) {
        std::cout << "使用合并排序" << std::endl;
        // 合并排序实现...
    }
};

// 使用排序策略
template <typename SortPolicyT>
class Collection : private SortPolicyT {
private:
    std::vector<int> data;
public:
    void add(int value) { data.push_back(value); }
    void sortData() {
        this->sort(data);  // 调用排序策略
    }
};

// 使用
Collection<QuickSort> collection1;
Collection<MergeSort> collection2;
```

## 7. CRTP优缺点

### 7.1 优点

- **性能优势**: 无虚函数调用开销，编译期解析
- **编译优化**: 更容易被内联和优化
- **类型安全**: 编译期类型检查
- **代码复用**: 不需要重复实现通用功能
- **静态接口检查**: 编译期发现接口不匹配问题

### 7.2 缺点

- **代码复杂性**: 使用模板和静态转换增加复杂度
- **编译时间**: 可能增加编译时间
- **代码体积**: 可能导致代码膨胀
- **运行时灵活性受限**: 不支持运行时类型变更
- **调试难度**: 模板代码通常难以调试
- **不支持异质集合**: 不能像虚函数那样处理不同类型对象

## 8. 最佳实践和使用建议

- 确保派生类实现基类所需的所有接口方法
- 避免在CRTP类层次结构中使用虚函数(混合使用会导致困惑)
- 当性能至关重要并且类型在编译期已知时，选择CRTP
- 需要运行时多态或处理异质对象集合时，使用虚函数
- 对性能关键代码，考虑使用CRTP替代虚函数
- 使用`static_assert`在编译期验证派生类接口
- 考虑将CRTP与其他模板技术结合使用
