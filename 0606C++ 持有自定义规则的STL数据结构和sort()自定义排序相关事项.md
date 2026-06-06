### C++ 持有自定义规则的STL数据结构和sort()自定义排序相关事项

#### Q: 为什么要传入 Lambda 实例初始化堆

**A: **

**因为 C++ 中的 Lambda 表达式默认没有默认构造函数（Default Constructor），`priority_queue` 无法在内部自己凭空创建一个 Lambda 对象，所以你必须从外部传一个现成的实例给它。**

##### 1. 以前的仿函数为什么不需要传实例？

```cpp
class cmp1 {
public:
    bool operator()(Info& a, Info& b) { ... }
};
priority_queue<Info, vector<Info>, cmp1> heap1; // 直接声明，没传参数
```

当你这样写时，`priority_queue` 在内部会这样创建比较器：

```cpp
cmp1 comparator; // 调用无参构造函数创建了一个 cmp1 对象
```

因为你写的 `class cmp1` 没有定义任何构造函数，C++ 编译器会自动为你生成一个**无参默认构造函数**，所以内部可以顺利创建。



##### 2. 现在的 Lambda 为什么必须传实例？

当你写下 Lambda 时：

```cpp
auto cmpCap = [](const Info& a, const Info& b) { return a.cap > b.cap; };
```

编译器在背后其实生成了一个**匿名的类**，大致等价于这样：

```cpp
class __UnnamedLambdaClass {
public:
    bool operator()(const Info& a, const Info& b) const { 
        return a.cap > b.cap; 
    }
private:
    // 隐藏规则：禁用了默认构造函数
    __UnnamedLambdaClass() = delete; 
};
```

**关键点来了**：C++ 标准规定，Lambda 生成的匿名类，其默认构造函数是被删除的（`= delete`）。

当 `priority_queue` 尝试在内部这样写时：

```cpp
decltype(cmpCap) comparator; // 错误！试图调用被删除的默认构造函数
```

编译器就会报错，因为它不知道怎么从零开始构造这个 Lambda 对象。



##### 3. `priority_queue` 的构造函数设计

`priority_queue` 的模板定义大致是：

```cpp
template<class T, class Container, class Compare>
class priority_queue {
private:
    Compare comp; // 内部持有一个比较器对象
public:
    // 关键构造函数：接收一个外部传入的比较器对象
    explicit priority_queue(const Compare& comp) : comp(comp) {}
};
```

既然它内部不能自己造一个 Lambda 对象，我们就必须通过构造函数，把我们外面创建好的那个 Lambda 实例 `cmpCap` **“塞”** 给它：

```cpp
// decltype(cmpCap) 解决了“类型是什么”的问题
// (cmpCap) 解决了“对象在哪里”的问题
priority_queue<Info, vector<Info>, decltype(cmpCap)> minCap(cmpCap);
```

这样，`priority_queue` 内部的 `comp` 就通过拷贝构造，拿到了你外部 `cmpCap` 的拷贝，问题迎刃而解。



##### 总结

- `decltype(cmpCap)` 仅仅是告诉编译器**类型**（就像图纸）。
- `cmpCap` 是一个实实在在的**对象**（就像按照图纸造出来的零件）。
- `priority_queue` 只有图纸造不出零件（因为没有默认构造函数），所以你必须直接把造好的零件 `(cmpCap)` 喂给它。





#### Q:sort自定义排序要求传入一个可调用对象，既然lambda没有构造函数，它为什么又是可调用的呢？sort的自定义和priority_queue自定义的区别在哪里？有没有跟priority_queue情况类似的stl数据结构库呢？

**A: **

##### 1. Lambda 没有默认构造函数，为什么又是可调用的？

这里有一个关键的概念区分：**“可调用”和“能否被构造”是两码事。**

Lambda 没有默认构造函数，意味着你**不能凭空创建**它。但是，当你写下 `auto cmp = [](int a, int b) { return a > b; };` 时，编译器已经在栈上帮你**构造出了一个 Lambda 实例 `cmp`**。

既然 `cmp` 已经是一个活生生的对象了，它内部重载了 `operator()`，你当然可以直接调用 `cmp(3, 5)`。所以，Lambda 是可调用的，前提是它已经被创建出来了。



##### 2. `sort` 和 `priority_queue` 自定义排序的区别在哪里？

这是最核心的问题。一句话概括：**`sort` 是算法（函数），它只需“借用”比较器；`priority_queue` 是容器（类），它需要“持有”比较器。**

##### `sort` 的情况（函数模板）

看 `sort` 的底层签名：

```cpp
template<class RandomIt, class Compare>
void sort(RandomIt first, RandomIt last, Compare comp);
```

`sort` 只是一个**函数**。当你在调用 `sort(vec.begin(), vec.end(), cmpCap)` 时，你把外部已经构造好的 Lambda 对象 `cmpCap` 作为参数**按值传递**给了 `comp`。 在 `sort` 函数执行的短暂期间，它使用这个传进来的 `comp` 对象来比较元素。函数执行完毕，`comp` 销毁。**`sort` 根本不需要自己去构造一个 `comp`，它是别人传什么，它就用什么。**

##### `priority_queue` 的情况（类模板）

看 `priority_queue` 的底层签名：

```cpp
template<class T, class Container, class Compare>
class priority_queue {
protected:
    Compare comp; // 关键：类内部有一个比较器成员变量！
    Container c;
public:
    priority_queue(const Compare& compare) : comp(compare) {} // 必须通过构造函数初始化
};
```

`priority_queue` 是一个**类**，它需要在对象的整个生命周期内保存这个比较规则，因为每次调用 `push`、`pop` 时都要进行堆调整，都需要用到 `comp`。

如果 `Compare` 是 Lambda：

1. 类内部声明了 `Compare comp;`。
2. 如果你不传参数，C++ 试图默认构造这个类时，会尝试写 `Compare comp;`，这就要求 `Compare` 必须有默认构造函数。
3. 但 Lambda 没有默认构造函数，所以编译失败。
4. 因此，你必须通过构造函数把外部的 Lambda 传进去，用初始化列表 `comp(compare)` 拷贝构造给内部的成员变量。



##### 3. 有没有跟 `priority_queue` 情况类似的 STL 数据结构？

**有！所有需要长期持有自定义规则的容器，情况都和 `priority_queue` 一模一样。**

最典型的就是**关联容器**，因为它们内部是红黑树，需要时刻根据比较规则来维护有序性。

##### 典型例子：`std::set` 和 `std::map`

如果你想用 Lambda 自定义 `std::set` 的排序规则，你必须像 `priority_queue` 一样，既传类型（`decltype`），又传实例。

```cpp
#include <iostream>
#include <set>
#include <functional>

int main() {
    auto cmp = [](int a, int b) { return a > b; }; // 降序排列的 Lambda

    // 错误写法：std::set<int, decltype(cmp)> mySet; 
    // 编译器报错：Lambda 没有默认构造函数！

    // 正确写法：和 priority_queue 一样，必须传入 Lambda 实例！
    std::set<int, decltype(cmp)> mySet(cmp);

    mySet.insert(10);
    mySet.insert(5);
    mySet.insert(20);

    for (int x : mySet) {
        std::cout << x << " "; // 输出 20 10 5
    }
    return 0;
}
```

##### 另一个例子：`std::unordered_set` 和 `std::unordered_map`

它们需要自定义哈希函数和相等判断谓词，如果用 Lambda，情况也完全一样：

```cpp
auto hash_func = [](int x) { return std::hash<int>()(x); };
auto eq_func = [](int a, int b) { return a == b; };

// 必须传入实例
std::unordered_set<int, decltype(hash_func), decltype(eq_func)> mySet(10, hash_func, eq_func);
```



##### 总结

- **可调用性**：Lambda 重载了 `()`，所以实例化后可调用；没有默认构造函数只是说不能凭空造出来而已。
- **算法 vs 容器**：`sort` 是一次性函数，你把 Lambda 实例当参数传进去用一下就完事了；`priority_queue` 是持久化的类，它要把 Lambda 作为成员变量存起来，所以必须通过构造函数传实例进去。
- **同类 STL**：凡是需要内部维持一种规则的容器（`set`, `map`, `unordered_set`, `unordered_map` 等），只要你想用 Lambda，都必须走 `decltype` 推导类型 + 传入实例初始化这条路。



#### Q: 怎么理解 不能凭空创建

**A: **

##### 1. 什么是“凭空创建”？

在 C++ 中，“凭空创建”指的是：**你手里只有一个类型的名字，没有任何外部数据，然后你直接声明一个该类型的变量。**

例如:

```cpp
int a;              // 凭空创建了一个 int，没问题，默认是随机值或 0
string s;           // 凭空创建了一个 string，没问题，默认是空字符串 ""
MyClass obj;        // 凭空创建了一个 MyClass，没问题，调用默认构造函数
```

如果我们试图对 Lambda 这样做：

```cpp
auto my_lambda = [](int a, int b) { return a > b; };

// 获取 Lambda 的类型名（编译器自动生成的，类似 __Lambda_12345）
using LambdaType = decltype(my_lambda);

// 试图“凭空创建”一个该类型的对象：
LambdaType another_lambda; // ❌ 编译报错！
```

这就是“不能凭空创建”的字面意思：**你不能只用类型名，不带任何参数地造出一个 Lambda 对象出来。**



##### 2. 为什么不能凭空创建？（核心原因：捕获状态）

Lambda 和普通的函数对象（仿函数）最大的不同，在于它能够**捕获外部变量**。

看下面这个例子：

```cpp
int threshold = 100;
auto my_lambda = [threshold](int a) { return a > threshold; };
```

在这个 Lambda 中，`threshold` 的值（100）被**打包塞进**了 `my_lambda` 这个对象里。这就是为什么我们叫它**闭包**——它不仅包含了行为的代码，还包含了运行时的环境数据。

现在，想象一下，如果 C++ 允许你“凭空创建”这个 Lambda：

```cpp
using LambdaType = decltype(my_lambda);
LambdaType empty_lambda; // 假设 C++ 允许这样做
```

问题来了：**`empty_lambda` 里的 `threshold` 值应该是多少？** 是 0？是随机值？还是必须去外面重新抓取？ 编译器无法替你做决定，因为 Lambda 的捕获逻辑是在你写 `[threshold]` 的那一瞬间确定的。如果允许凭空默认构造，就会造出一个“状态残缺”的怪物，这违背了 C++ 的安全原则。

**所以，C++ 标准索性规定：Lambda 表达式生成的闭包类型，删除了默认构造函数。** 你不能凭空造它，你只能通过**写下方括号 `[]` 的那一瞬间**，让编译器帮你把环境和代码打包成一个对象。



##### 3. 一个有趣的特例（C++20 的妥协）

如果你的 Lambda **没有任何捕获**（方括号里是空的 `[]`），它就不包含任何打包进来的外部数据。这种情况下，按理说应该是可以凭空创建的。

事实上，C++20 之前，即使没有捕获，标准也不允许凭空创建。但 C++20 放宽了这个限制：

```cpp
auto empty_lambda = [](int a, int b) { return a > b; }; // 无捕获

using LambdaType = decltype(empty_lambda);

// C++20 之前：❌ 编译错误
// C++20 及之后：✅ 编译通过！
LambdaType another_lambda; 
```

**为什么 C++20 允许了？** 因为无捕获的 Lambda 不携带任何外部状态，它本质上退化为一个纯粹的函数。既然没有状态需要初始化，默认构造它就是安全的。

##### 总结

“不能凭空创建”，是因为 Lambda 的灵魂在于**捕获状态**。如果允许不提供任何信息就把它造出来，它内部捕获的那些变量就变成了无源之水。因此，C++ 强制要求：**Lambda 必须在你写下 `[](...){}` 的时候由编译器生成，或者通过拷贝已有 Lambda 来创建，绝不能只用一个类型名就凭空声明。**