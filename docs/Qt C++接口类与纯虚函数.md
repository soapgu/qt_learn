# Qt C++ 接口类与纯虚函数

## 1. 文档目标

本文以 Caliburn.Micro.Qt 框架的 `IWindowManager`、`IParent`、`IChild`、`IConductor` 等真实接口为主线，解释 C++ 如何用抽象类模拟接口、Qt 对接口机制的两层增强，以及多态基类的析构函数写法。

重点回答：

- `virtual bool busy() const = 0;` 里每个关键字分别是什么意思；
- C# 的 `interface` 在 C++ 里如何对应，为什么 C++ 没有单独的 interface 关键字也能做接口；
- `Q_DECLARE_INTERFACE` 与 `Q_INTERFACES` 到底做了什么，`qobject_cast<IConductor *>()` 为什么能转换一个不继承 QObject 的接口；
- `virtual ~IParent() = default;` 和 `~WindowManager() override;` 各自适用什么场景，什么时候必须手写析构函数；
- 哪些笔误会静默编译通过却在运行时出问题。

前置阅读：接口类与 `Q_OBJECT`、moc、`qt_metacast` 的关系参见 [Qt 元对象系统与 QML 类型注册](Qt元对象系统与QML类型注册.md)；接口方法如何暴露给 QML 调用参见 [Qt Q_INVOKABLE 与跨平台命令调用对照](Qt%20Q_INVOKABLE与跨平台命令调用对照.md)。

官方参考：

- [Qt：Q_DECLARE_INTERFACE](https://doc.qt.io/qt-6/qobject.html#Q_DECLARE_INTERFACE)
- [Qt：qobject_cast](https://doc.qt.io/qt-6/qobject.html#qobject_cast)
- [Qt：The Meta-Object System](https://doc.qt.io/qt-6/metaobjects.html)
- [C++ Core Guidelines: A destructor shall not fail](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Rc-dtor-noexcept)
- [C++ Core Guidelines: Make base class destructors public and virtual](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Rc-dtor-virtual)

## 2. 从 C# interface 到 C++ 抽象类

Caliburn.Micro 的 .NET 版本里，窗口服务是一个接口。WPF/C# 的写法：

```csharp
public interface IWindowManager
{
    bool Busy { get; }
    Task<bool?> ShowDialogAsync(object viewModel, object owner);
}
```

C++ 没有专门的 interface 关键字，惯用手法是"只含纯虚函数的抽象类"。Caliburn.Micro.Qt 的对应声明：

```cpp
// 摘自 Caliburn.Micro.Qt 的 IWindowManager.h，略有精简。
class IWindowManager : public QObject
{
    Q_OBJECT
    Q_PROPERTY(bool busy READ busy NOTIFY busyChanged)

public:
    virtual bool busy() const = 0;
    virtual QFuture<DialogResult> showDialogAsync(std::unique_ptr<ScreenViewModel> viewModel,
                                                 QObject *requester) = 0;
    virtual void cancelDialogsFor(QObject *requester) = 0;
};
```

### 2.1 逐关键字拆解

以 `virtual bool busy() const = 0;` 为例：

```cpp
virtual bool busy() const = 0;
// ┬─┬────┬────┬───┬─┬─┬
// │ │    │    │   │ │ └─ "= 0"：纯虚指定符，本类不提供实现
// │ │    │    │   │ └─── 分号
// │ │    │    │   └───── "const"：常量成员函数限定
// │ │    │    └───────── 参数列表（空）
// │ │    └────────────── 函数名
// │ └─────────────────── 返回类型
// └───────────────────── "virtual"：支持运行时多态
```

四个部分各自承担一个职责：

| 关键字 | 含义 | 作用 |
| --- | --- | --- |
| `virtual` | 虚函数 | 通过基类指针/引用调用时，实际执行派生类重写的版本 |
| `const` | 常量成员函数 | 承诺不修改成员变量；在 `const` 指针上也可调用 |
| `= 0` | 纯虚指定符 | 基类不实现，只声明契约；含纯虚函数的类是抽象类 |
| `override`（派生类） | 显式重写 | 让编译器检查"确实在重写基类虚函数"，防拼写错误 |

`const` 特别容易忽视的一点：**它是函数签名的一部分**。派生类重写时必须同样带 `const`，否则是定义了一个新函数而不是重写：

```cpp
class IWindowManager : public QObject
{
public:
    virtual bool busy() const = 0;
};

class WindowManager : public IWindowManager
{
public:
    bool busy() const override { return m_request != nullptr; } // ✅ 正确重写
    // bool busy() override { ... }   // ❌ 缺 const：不是重写，编译报错
    //                                  // （加了 override 才能被发现；不加则是隐藏）
};
```

`= 0` 的直接后果：抽象类不能实例化，派生类必须实现全部纯虚函数才能创建对象：

```cpp
IWindowManager *p = nullptr;
IWindowManager wm;   // ❌ 编译错误：abstract class cannot be instantiated
WindowManager impl;  // ✅ 具体子类
```

一个冷知识：`= 0` 的函数也可以有函数体（在类外定义），派生类可以显式调用。这用于"有共同默认实现但仍强制派生类表态"的场景。Caliburn.Micro.Qt 没有使用这个特性，日常也少见，知道即可。

### 2.2 为什么 C++ 选择抽象类而非 interface 关键字

C++ 的设计哲学是不为语言增加只能做一件事的关键字。"纯虚函数集合 + 无状态 + 虚析构"已经完整表达了接口语义，而且比 interface 更灵活——同一个抽象类里可以混放纯虚函数和带默认实现的虚函数，接口与抽象基类之间是连续过渡而非二选一。

代价是把纪律交给作者：编译器不强制"接口类无状态"，需要自己约束。

## 3. Qt 接口类的两种写法

Caliburn.Micro.Qt 里并存两种接口形态，分别解决不同问题。

### 3.1 纯 C++ 接口：不继承 QObject

`IParent`、`IChild`、`IConductor` 是纯 C++ 接口，不依赖任何 Qt 基类：

```cpp
// IParent.h —— 纯 C++ 接口
class IParent
{
public:
    virtual ~IParent() = default;
    virtual QList<ViewModelBase *> getChildren() const = 0;
};

Q_DECLARE_INTERFACE(IParent, "Caliburn.Micro.Qt.IParent/1.0")
```

```cpp
// IConductor.h —— 接口继承接口
class IConductor : public IParent
{
public:
    ~IConductor() override = default;
    virtual bool activateItem(ViewModelBase *item) = 0;
    virtual bool deactivateItem(ViewModelBase *item, bool close) = 0;
};

Q_DECLARE_INTERFACE(IConductor, "Caliburn.Micro.Qt.IConductor/1.0")
```

实现类是 QObject，通过 `Q_INTERFACES` 声明自己实现了哪些接口：

```cpp
// ConductorBase.h —— QObject 实现类
class ConductorBase : public ScreenViewModel, public IConductor
{
    Q_OBJECT
    Q_INTERFACES(IParent IConductor)   // 告诉 moc：我实现了这两个接口

public:
    QList<ViewModelBase *> getChildren() const override;
    bool activateItem(ViewModelBase *item) override;
    bool deactivateItem(ViewModelBase *item, bool close) override;
};
```

这套机制最实际的收益：一个 Screen 可以向"逻辑父级"请求关闭，而不需要知道对方的具体类型，甚至不需要对方是某种 QObject 子类：

```cpp
// ScreenViewModel::tryClose() —— 跨接口调用实例
bool ScreenViewModel::tryClose()
{
    auto *conductor = qobject_cast<IConductor *>(parentViewModel());
    return conductor && conductor->deactivateItem(this, true);
}
```

### 3.2 抽象 QObject 接口：需要信号和属性时

`IWindowManager` 要向 QML 暴露 `busy` 属性和 `busyChanged` 信号，而信号只有 QObject 体系才有。所以它继承 QObject，让接口本身携带元对象能力：

```cpp
// IWindowManager.h —— 抽象 QObject 接口（摘录）
class IWindowManager : public QObject
{
    Q_OBJECT
    Q_PROPERTY(bool busy READ busy NOTIFY busyChanged)

signals:
    void busyChanged();

public:
    using QObject::QObject;
    virtual bool busy() const = 0;
    virtual ScreenViewModel *currentDialog() const = 0;
    virtual void closeDialog(ScreenViewModel *viewModel, DialogResult result = std::nullopt) = 0;
};
```

`WindowManager` 作为唯一具体子类，override 全部纯虚函数并补上实现：

```cpp
class WindowManager : public IWindowManager
{
    Q_OBJECT

public:
    explicit WindowManager(QObject *parent = nullptr);
    ~WindowManager() override;                    // 手写析构，见第 5 节

    bool busy() const override { return bool(m_request) || m_finishing; }
    void closeDialog(ScreenViewModel *viewModel, DialogResult result = std::nullopt) override;
    // ...
};
```

注意 `IWindowManager` 没有声明析构函数：它继承 QObject，而 `QObject::~QObject()` 已经是 virtual，隐式生成的析构自动继承虚性。零状态接口无事可做，不写是合理的。

### 3.3 两种写法的选择

| 维度 | 纯 C++ 接口（IParent） | 抽象 QObject（IWindowManager） |
| --- | --- | --- |
| 接口能否声明 signal / Q_PROPERTY | ❌ 不能 | ✅ 能 |
| 能否参与 qobject_cast 接口转换 | ✅ 需 Q_DECLARE_INTERFACE | ✅ 直接走 QObject 继承链 |
| 实现类约束 | 必须是 QObject + Q_INTERFACES | 必须是 QObject 子类 |
| 接口头文件是否依赖 Qt | 只依赖容器等工具头 | 依赖 QObject、moc |
| 典型场景 | 纯 C++ 协议（生命周期、能力查询） | 需要向 QML 暴露状态/通知的服务契约 |

经验法则：**接口需要信号或属性时才继承 QObject**，否则选纯 C++ 接口——依赖更少，也能被非 QObject 类实现（虽然实践中 Qt 项目里实现方几乎总是 QObject）。

## 4. Q_DECLARE_INTERFACE 与 Q_INTERFACES 的机制

`qobject_cast<IConductor *>(someQObject)` 能把一个 QObject 转成"它根本不继承的接口指针"，靠的是 moc 生成的一段字符串匹配代码。理解机制后，"为什么忘写 Q_INTERFACES 就转换失败"就不再神秘。

### 4.1 三个宏各做什么

```cpp
Q_DECLARE_INTERFACE(IConductor, "Caliburn.Micro.Qt.IConductor/1.0")
```

在全局作用域声明接口的 **IID（接口标识字符串）**，并把 IID 与接口类型关联起来。IID 命名约定是"反向域名/版本号"，版本号用于将来接口不兼容变更时升版。这个宏不依赖 moc，写在接口头文件末尾。

```cpp
Q_INTERFACES(IParent IConductor)
```

写在实现类的 `Q_OBJECT` 之后，告诉 moc："生成 `qt_metacast` 时，为这两个接口各加一个转换分支。"

moc 为 `ConductorBase` 生成的代码大致是：

```cpp
// moc 生成代码（示意，简化）
void *ConductorBase::qt_metacast(const char *_clname)
{
    if (!_clname)
        return nullptr;
    if (strcmp(_clname, "ConductorBase"))
        return static_cast<void *>(this);
    // Q_INTERFACES(IParent IConductor) 生成的分支：
    if (strcmp(_clname, qt_incomplete_metaTypeid<IParent *>(...)))
        return static_cast<IParent *>(this);          // 接口不在继承链上，
    if (strcmp(_clname, ...))                          // static_cast 走多继承偏移
        return static_cast<IConductor *>(static_cast<IParent *>(this));
    return ScreenViewModel::qt_metacast(_clname);
}
```

`static_cast<IParent *>(this)` 这一步是整个机制的核心：C++ 编译器知道多继承布局下接口子对象的偏移，moc 只需要"在运行时按名字触发这次转换"。

### 4.2 qobject_cast 的调用链

```text
qobject_cast<IConductor *>(obj)
    → obj->qt_metacast(IConductor 的 IID)      // 沿 QObject 父类链逐层询问
        → ConductorBase::qt_metacast 命中 "IConductor"
            → static_cast<IConductor *>(this)   // 编译期已知的偏移
            → 返回接口指针
    → 命中失败（没有任何一层实现声明该接口）
        → 返回 nullptr
```

所以两个宏是一对：`Q_DECLARE_INTERFACE` 定义"这个名字（IID）"，`Q_INTERFACES` 让 moc 在实现类的 `qt_metacast` 里登记"这个名字可以转到我"。漏掉任何一个，`qobject_cast` 都静默返回 nullptr——这也是第 6 节陷阱清单的第一名。

### 4.3 与 dynamic_cast 的对比

C++ 自带的 `dynamic_cast<IConductor *>(obj)` 也能做跨接口转换（要求实现类是多态类型），且不需要任何 Qt 宏。Caliburn.Micro.Qt 选择 qobject_cast 的原因：

| 维度 | qobject_cast | dynamic_cast |
| --- | --- | --- |
| 前提 | 类型有 Q_OBJECT、接口已声明 | 类有多态（有虚函数） |
| 跨模块（插件边界） | ✅ 只比较 IID 字符串 | ❌ 依赖编译单元一致的 RTTI 名字 |
| 开销 | 一次字符串比较链 | RTTI 查找 |
| 接口可否不继承 QObject | ✅ | ✅ |

Qt 插件体系（QPluginLoader）里对象来自动态库，RTTI 名字跨编译器不可靠，IID 字符串是唯一稳定标识——这就是 Qt 不用 dynamic_cast 而自建一套的原因。

## 5. 虚析构函数的写法

多态基类的析构函数有一条铁律：**通过基类指针 delete 派生类对象时，析构必须是虚的**，否则只执行基类析构，派生类部分不析构——未定义行为：

```cpp
IParent *p = new ConductorBase();  // 假设 ConductorBase 持有堆资源
delete p;                           // 若 ~IParent() 非虚：只调 ~IParent()，资源泄漏
```

在这个前提下，实际会遇到三种写法。

### 5.1 接口层：`virtual ~IParent() = default;`

```cpp
class IParent
{
public:
    virtual ~IParent() = default;
    // ...
};
```

`= default` 表示"编译器生成默认实现"，写出来只为把 `virtual` 显式钉在析构上。它等价于什么都不写时编译器给出的版本，区别是意图可见。Caliburn.Micro.Qt 的 `IParent`、`IChild`、`IConductor` 都用这个写法。

`IWindowManager` 则连这行都没写——它继承 QObject，`QObject::~QObject()` 已是 virtual，隐式析构继承虚性。两种"不做事"的差异只在"是否已经从基类继承虚析构"。

### 5.2 实现层：`~WindowManager() override;`

```cpp
class WindowManager : public IWindowManager
{
public:
    ~WindowManager() override;   // 声明
};

// WindowManager.cpp
WindowManager::~WindowManager()
{
    m_destroying = true;
    if (m_request) {
        complete(std::nullopt);
        finish();
    }
}
```

两个语法点先澄清：

- **override 可以修饰析构函数**：基类析构是 virtual，派生析构自动也是虚函数并构成"重写"，`override` 让编译器帮你确认这一点；
- **`= default` 与手写不是风格偏好之差**：`= default` 的含义是"逐个销毁成员就够了"，手写的适用条件是"销毁前还有协议级动作要做"。

`WindowManager` 必须手写，因为它析构时可能**持有在途的异步请求**：

```cpp
struct WindowManager::Request
{
    QPromise<DialogResult> promise;   // 调用方正拿着对应的 QFuture 等结果
    // ...
};
```

如果析构写成 `= default`，`m_request` 被直接销毁：

- `QPromise` 永远不会 `finish()`；
- 调用方的 `QFuture` 永不完成，`.then(...)` 回调永不执行；
- 发起方 ViewModel 的 `resetPending` 卡在 true，按钮永久禁用——一次静默的异步泄漏，没有任何报错。

所以析构函数的任务是"按协议收尾"而非"释放内存"：置 `m_destroying` 阻止新请求 → `complete(std::nullopt)` 把当前请求标记为"无决定关闭" → `finish()` 断开监听、执行弹窗 VM 的完整关闭生命周期、完成 promise 让 Future 落地。

判断标准可以概括成一句话：**成员变量全都被动销毁就够，用 `= default`；对象消亡前需要对外部世界（Future、回调、注册表、连接）做最后一次通知，手写析构**。

### 5.3 写法速查

| 场景 | 写法 | 示例 |
| --- | --- | --- |
| 纯 C++ 接口 | `virtual ~I() = default;` | IParent、IChild、IConductor |
| 继承 QObject 的零状态接口 | 不写（QObject 析构已虚） | IWindowManager |
| 持有运行时资源的实现类 | 手写 `~Impl() override;` | WindowManager |
| 非多态工具类 | 不写 virtual，甚至 `= delete` 阻止继承 | — |

## 6. 常见陷阱清单

以下问题多数能编译通过，只在运行时暴露，值得在 code review 里逐条对照：

1. **重写漏了 `const`**：基类 `virtual bool busy() const = 0;`，派生类写 `bool busy() override`。加了 `override` 会编译报错（推荐）；没加则是定义了新函数并隐藏基类版本，通过基类指针调用的仍是纯虚声明——链接期或运行期才炸。
2. **实现类忘写 `Q_INTERFACES`**：编译正常，`qobject_cast<IParent *>(obj)` 恒返回 nullptr，逻辑分支静默走不到。接口声明与实现分头提交时最容易漏。
3. **IID 冲突或写错**：两个不同接口声明了相同 IID，`qt_metacast` 按字符串匹配，转换到错误类型是未定义行为。IID 应包含项目前缀（如 `Caliburn.Micro.Qt.`）。
4. **接口类混入状态成员**：接口被迫写拷贝语义、实现类多继承时状态分叉。接口应零状态；需要默认实现就放普通虚函数或基类。
5. **多态基类析构忘记虚化**：纯 C++ 接口漏写 `virtual ~I() = default;`，通过接口指针 delete 实现对象即 UB。QObject 子类接口无此问题。
6. **接口方法声明成 Q_INVOKABLE**：QML 里拿到的往往是具体 VM 或服务对象，接口本身不向 QML 注册；把 Q_INVOKABLE 留给具体类型，接口保持纯 C++ 契约。Caliburn.Micro.Qt 的接口方法全部是普通虚函数。
7. **析构函数里做复杂协议收尾时忘记异常**：析构默认 noexcept，收尾代码抛异常即 `std::terminate`。`WindowManager` 的收尾路径全部是 noexcept 安全操作；若你的析构需要调用可能抛异常的代码，用 try/catch 包住并记录，而不是让异常逃逸。

## 7. 总结

- C++ 用"纯虚函数集合 + 虚析构"的抽象类模拟 interface：`virtual` 提供多态，`= 0` 强制派生类实现，`const` 是签名的一部分，重写必须原样携带。
- Qt 接口有两层形态：纯 C++ 接口（`IParent` 风格，配 Q_DECLARE_INTERFACE）适合纯协议；抽象 QObject（`IWindowManager` 风格）在需要 signal/Q_PROPERTY 时使用。
- `Q_DECLARE_INTERFACE` + `Q_INTERFACES` 是 IID 字符串与 moc 生成的 `qt_metacast` 分支的配对机制，`qobject_cast` 靠它完成"不继承 QObject 的接口"的运行时转换；两者缺一转换即静默失败。
- 虚析构铁律：多态基类析构必须虚。零状态用 `= default` 显式表达，QObject 派生接口可以不写；实现类在析构时若持有在途异步状态（QPromise、连接、注册关系），必须手写析构按协议收尾——`= default` 的后果不是崩溃，而是外部世界永远等不到结果。
- 接口设计的纪律：零状态、方法只声明契约、IID 带项目前缀、实现类用 override 让编译器当审稿人。
