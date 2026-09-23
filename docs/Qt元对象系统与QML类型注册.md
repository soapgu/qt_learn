# Qt 元对象系统与 QML 类型注册

## 1. 文档目标

本文以 Qt 6.8.3 和当前表决系统为背景，集中说明：

- 普通 C++ 类型为什么不能直接满足 Qt 的运行时查询和 QML 绑定需求；
- `moc`、`Q_OBJECT`、`Q_GADGET` 和 `Q_ENUM` 分别解决什么问题；
- `Q_PROPERTY`、signal、状态和瞬时事件之间是什么关系；
- `QML_ELEMENT`、`QML_NAMED_ELEMENT`、`QML_UNCREATABLE` 如何将 C++ 类型注册到 QML；
- `VoteServiceState` 与 `VoteServiceStateQml` 为什么分成两个类型；
- 当前麒麟崩溃对应的错误注册方式，以及推荐修复方式。

属性绑定的完整细节参见：[Qt Q_PROPERTY 与跨平台双向绑定对照](Qt%20Q_PROPERTY与跨平台双向绑定对照.md)。QML 组件事件和后端事件的分层参见：[Qt QML Signal 与分层事件流](Qt%20QML%20Signal与分层事件流.md)。

官方参考：

- [Qt Meta-Object System](https://doc.qt.io/qt-6.8/metaobjects.html)
- [The Property System](https://doc.qt.io/qt-6.8/properties.html)
- [QMetaEnum](https://doc.qt.io/qt-6.8/qmetaenum.html)
- [Defining QML Types from C++](https://doc.qt.io/qt-6.8/qtqml-cppintegration-definetypes.html)
- [QML and C++ Type Registration](https://doc.qt.io/qt-6.8/qtqml-cppintegration-definetypes.html#registering-an-instantiable-object-type)

## 2. 普通 C++ 与 Qt 元对象系统

普通 C++ 类可以定义成员、方法和枚举：

```cpp
class ServerState
{
public:
    enum State {
        Stopped,
        Listening
    };

    State state() const;
};
```

但 Qt 和 QML 不能仅凭这个声明完成下列运行时查询：

```text
对象是否有名为 state 的属性？
属性应调用哪个 Getter？
属性变化时应监听哪个信号？
对象有哪些可调用方法？
枚举数字 1 对应哪个名称？
这个 C++ 类型在 QML 模块中叫什么？
```

Qt 使用元对象系统补充这些信息。构建时，Meta-Object Compiler（`moc`）扫描带有 Qt 元对象宏的头文件，并生成额外的 C++ 代码。常见产物包括：

- 类的 `staticMetaObject`；
- 属性、方法、signal、slot 和枚举元数据；
- signal 激活和动态方法调用需要的代码；
- 供 `qmltyperegistrar` 继续生成 QML 类型注册信息的元数据。

因此这些宏不是注释，也不是单纯的代码提示。它们会实际参与代码生成、编译和链接。

## 3. `Q_OBJECT`：完整的 Qt 对象能力

`Q_OBJECT` 用于继承自 `QObject` 的类：

```cpp
class VoteSessionService final : public QObject
{
    Q_OBJECT

public:
    explicit VoteSessionService(QObject *parent = nullptr);

signals:
    void runningChanged(bool running);
};
```

它提供完整的 Qt 元对象能力：

- signals 和 slots；
- `Q_PROPERTY` 及其 `NOTIFY` 通知；
- `Q_INVOKABLE` 动态调用；
- `QMetaObject` 运行时查询；
- `qobject_cast`；
- QObject 父子对象生命周期；
- C++ 对象与 QML 的交互基础。

### 3.1 signal、slot 和普通方法

signal 表示一次通知：

```cpp
signals:
    void controllerConnected(const QString &ip);
```

发送 signal：

```cpp
emit controllerConnected(ip);
```

slot 或普通成员函数都可以作为 `connect()` 的接收端。`slots` 关键字主要把方法加入元对象信息；现代函数指针形式的 `connect()` 也可以连接普通成员函数。

signal 本身不保存当前值。如果发送时没有接收者，这次事件不会被自动保留。当前状态应存放在成员变量或 Model 中，再通过属性对外公开。

### 3.2 `Q_INVOKABLE`

```cpp
Q_INVOKABLE void startServer();
```

`Q_INVOKABLE` 将方法登记到元对象系统，使 QML 或 `QMetaObject::invokeMethod()` 能按元数据调用它。普通 `public` C++ 方法不会因此自动对 QML 可调用。

### 3.3 QObject 生命周期

QObject 可以使用父子关系管理生命周期：

```cpp
m_eventLogModel = new EventLogModel(this);
```

父对象销毁时会销毁子对象。该机制只属于 QObject；`Q_GADGET` 类型没有父子对象生命周期。

## 4. `Q_GADGET`：普通类型的轻量元数据

`Q_GADGET` 用于不继承 `QObject`，但需要部分 Qt 元数据能力的普通类型：

```cpp
class VoteServiceState final
{
    Q_GADGET

public:
    enum State {
        Stopped,
        Starting,
        Listening,
        Connected,
        Stopping,
        Error
    };
    Q_ENUM(State)
};
```

适合使用 `Q_GADGET` 的类型包括：

- 枚举容器；
- 值对象；
- DTO 和配置结构；
- 不需要 signal 和对象生命周期的数据类型。

`Q_GADGET` 可以提供枚举和属性元数据，但它不是 QObject。它不支持：

- signals 和 slots；
- `qobject_cast`；
- QObject 父子生命周期；
- 依赖 `NOTIFY` signal 的响应式属性通知。

不要为了少写一个轻量类型而让所有数据对象都继承 `QObject`；也不要在需要 signal 和生命周期时用 `Q_GADGET` 冒充 QObject。

## 5. `Q_ENUM`：把枚举加入元对象

普通 C++ 枚举只提供编译期名称和值。`Q_ENUM` 将枚举登记到所属类的 Qt 元对象：

```cpp
class Example
{
    Q_GADGET

public:
    enum State {
        Stopped,
        Running
    };
    Q_ENUM(State)
};
```

登记后可以查询名称和值：

```cpp
const QMetaEnum metaEnum = QMetaEnum::fromType<Example::State>();
const char *name = metaEnum.valueToKey(Example::Running);
const int value = metaEnum.keyToValue("Running");
```

`Q_ENUM` 常用于：

- 调试和日志中显示枚举名称；
- `QVariant` 和 Qt 元类型系统识别枚举；
- signal/slot 参数携带枚举；
- QML 类型注册后访问枚举常量。

`Q_ENUM` 必须依附于带有 `Q_OBJECT` 或 `Q_GADGET` 的类。命名空间对应的是 `Q_NAMESPACE` 和 `Q_ENUM_NS`，不能在没有元对象载体的普通类中单独使用 `Q_ENUM`。

## 6. 三个宏如何选择

| 需求 | `Q_OBJECT` | `Q_GADGET` | `Q_ENUM` |
| --- | --- | --- | --- |
| 继承 QObject | 必须 | 不需要 | 不决定 |
| signals/slots | 支持 | 不支持 | 不负责 |
| `Q_PROPERTY + NOTIFY` | 支持 | 不支持完整通知链 | 不负责 |
| `Q_INVOKABLE` | 支持 | 不作为对象调用入口 | 不负责 |
| QObject 父子生命周期 | 支持 | 不支持 | 不负责 |
| 轻量枚举或值类型元数据 | 可以但通常过重 | 适合 | 登记具体枚举 |
| 枚举名称和值反射 | 配合 `Q_ENUM` | 配合 `Q_ENUM` | 负责登记 |

选择流程：

```text
类型是否需要 signal、属性通知、QML 对象调用或 QObject 生命周期？
├── 是：继承 QObject，使用 Q_OBJECT
└── 否：是否需要 Qt 枚举或值类型元数据？
    ├── 是：使用 Q_GADGET
    └── 否：保持普通 C++ 类型

类型内部的枚举是否需要进入 Qt 元对象系统？
├── 是：增加 Q_ENUM
└── 否：保留普通 enum/enum class
```

## 7. `Q_PROPERTY` 不是成员变量

下面的宏只声明元对象属性：

```cpp
Q_PROPERTY(bool running READ running NOTIFY runningChanged)
```

它不会自动生成：

- `m_running` 成员变量；
- `running()` Getter；
- `runningChanged()` signal；
- 修改属性和发送通知的代码。

完整实现仍需显式编写：

```cpp
class Example : public QObject
{
    Q_OBJECT
    Q_PROPERTY(bool running READ running NOTIFY runningChanged)

public:
    bool running() const { return m_running; }

signals:
    void runningChanged();

private:
    void setRunning(bool running)
    {
        if (m_running == running) {
            return;
        }
        m_running = running;
        emit runningChanged();
    }

    bool m_running = false;
};
```

常用字段含义：

| 字段 | 作用 |
| --- | --- |
| `READ` | 指定 Getter |
| `WRITE` | 指定 Setter；省略时对外只读 |
| `NOTIFY` | 指定属性变化通知 signal |
| `CONSTANT` | 表示对象生命周期内不变化 |

QML Binding 订阅 `NOTIFY` signal。收到通知后，QML 会重新调用 `READ` Getter，再重新计算依赖该属性的表达式。

## 8. 当前状态与瞬时事件

需要区分“现在是什么”和“刚刚发生了什么”。

当前状态应使用成员变量和属性：

```cpp
Q_PROPERTY(bool running READ running NOTIFY runningChanged)
```

瞬时事件使用普通 signal：

```cpp
signals:
    void controllerConnected(const QString &ip);
    void errorOccurred(const QString &context, const QString &message);
```

在当前项目中：

```text
running/serviceState/controllerIp
    = 当前状态，可随时读取

controllerConnected/errorOccurred
    = 某个时刻发生的事件，本身不保存历史

EventLogModel
    = 将重要事件保存为可回看的历史
```

不要把所有 signal 都包装成属性，也不要只发 signal 而不保存界面需要随时读取的当前状态。

## 9. 元对象宏与 QML 注册宏是两层机制

`Q_OBJECT`、`Q_GADGET` 和 `Q_ENUM` 负责 Qt 元对象信息。它们不等于自动注册为 QML 类型。

QML 注册宏负责另一层工作：

| 宏 | 作用 |
| --- | --- |
| `QML_ELEMENT` | 使用 C++ 类型名注册到当前 QML 模块 |
| `QML_NAMED_ELEMENT(Name)` | 使用指定名称注册到 QML 模块 |
| `QML_UNCREATABLE(Reason)` | QML 能看到类型和枚举，但不能创建实例 |
| `QML_SINGLETON` | 将类型注册为 QML 单例 |

例如 ViewModel：

```cpp
class VoteServerViewModel final : public QObject
{
    Q_OBJECT
    QML_ELEMENT
    QML_UNCREATABLE("由应用程序创建")
};
```

这里分别表示：

```text
Q_OBJECT
    → 生成 QObject 元对象能力

QML_ELEMENT
    → 在 VoteDevice 模块中注册 VoteServerViewModel 类型

QML_UNCREATABLE
    → QML 可将它用于强类型属性，但不能自行 new 一个实例
```

因此 `Main.qml` 可以声明：

```qml
required property VoteServerViewModel voteViewModel
```

实际实例仍由 C++ 组合根创建并注入。

## 10. 静态 QML 模块的构建链

当前项目使用：

```cmake
qt_add_library(vote_ui STATIC)

qt_add_qml_module(vote_ui
    URI VoteDevice
    SOURCES
        src/presentation/EventLogModel.cpp
        src/presentation/VoteServerViewModel.cpp
        src/presentation/VoteServiceStateQml.h
    QML_FILES
        qml/Main.qml
        qml/components/EventLogPanel.qml
)
```

主要工具职责如下：

```text
moc
    → 生成 QObject/Q_GADGET 元对象代码和元数据

qmltyperegistrar
    → 根据元数据生成 VoteDevice 模块的 C++ 类型注册代码和 qmltypes 信息

rcc
    → 将 QML、qmldir 等资源编译进程序

qmlcachegen
    → 为 QML 生成缓存或编译产物
```

静态 QML 模块还会生成资源初始化目标，例如：

```text
vote_ui_resources_1
vote_ui_resources_2
vote_ui_resources_3
```

这些是 CMake/Qt 自动生成的对象库，用于把模块元数据、QML 文件和额外 `qmldir` 资源链接进应用，不是额外的业务项目。

## 11. 当前项目中的类型选择

### 11.1 `VoteSessionService`

它需要：

- 接收后端事件；
- 发出状态、连接、断开和错误 signal；
- 参与 QObject 生命周期和连接自动断开。

因此使用：

```cpp
class VoteSessionService final : public QObject
{
    Q_OBJECT
};
```

### 11.2 `VoteServerViewModel`

它需要：

- 通过 `Q_PROPERTY` 向 QML 公开状态；
- 通过 `NOTIFY` 驱动 Binding；
- 通过 `Q_INVOKABLE` 接收界面命令；
- 作为强类型属性出现在 QML。

因此使用：

```cpp
Q_OBJECT
QML_ELEMENT
QML_UNCREATABLE(...)
```

### 11.3 `EventLogModel`

它继承 `QAbstractListModel`，需要 Model/View 通知、QObject 生命周期和 QML roles，因此使用 `Q_OBJECT`。

### 11.4 `VoteServiceState`

它只是项目自有状态枚举，不发 signal，也不需要对象生命周期，因此使用：

```cpp
Q_GADGET
Q_ENUM(State)
```

真正的业务状态只保存为 `VoteServiceState::State`。Service 维护状态，ViewModel 只读取并转换成展示文字。

## 12. 为什么存在 `VoteServiceStateQml`

`VoteServiceStateQml` 不是第二套状态机，也不保存状态。它只负责让 QML 能使用具名枚举常量：

```qml
VoteServiceState.Starting
VoteServiceState.Listening
VoteServiceState.Error
```

领域层的 `VoteServiceState` 不直接包含 QML 注册宏，目的是保持依赖方向：

```text
presentation → domain
```

而不是让领域层反向依赖 QML。适配器的 C++ 名称可以是 `VoteServiceStateQml`，但通过 `QML_NAMED_ELEMENT(VoteServiceState)` 暴露给 QML 后，QML 只看到 `VoteServiceState`。

适配器枚举值显式引用领域枚举值：

```cpp
Starting = VoteServiceState::Starting
```

这表示导出同值常量，不表示重新保存或转换业务状态。

拆分适配器还可以避免同一个 `Q_GADGET` 头文件同时被 `vote_domain` 和 `vote_ui` 的 AUTOMOC 处理，进而产生重复的 `staticMetaObject` 链接符号。

## 13. 麒麟崩溃：当前错误实现

截至本文编写时，仓库中的适配器仍是下面这种形式，尚未修复：

```cpp
struct VoteServiceStateQml
{
    Q_GADGET
    QML_NAMED_ELEMENT(VoteServiceState)
    QML_UNCREATABLE("仅用于公开服务状态枚举")
};
```

麒麟运行时输出：

```text
qt.qml.typeregistration: Invalid QML element name "VoteServiceState";
value type names should begin with a lowercase letter
free(): invalid pointer
```

原因是 Qt 将带 QML 注册信息的 `Q_GADGET` 视作 QML 值类型，而值类型名称必须以小写字母开头。`VoteServiceState` 是大写对象类型风格的名称，因此注册失败；后续 `free(): invalid pointer` 导致应用崩溃。

这不是供应商 SDK、端口监听或控制器事件导致的错误，而是在 QML 类型注册期间发生的错误。

## 14. 推荐修复实现

当前需求是公开一个大写名称的枚举容器，而不是公开可实例化的值类型。推荐将适配器改为不可创建的 QObject 类型：

```cpp
class VoteServiceStateQml : public QObject
{
    Q_OBJECT
    QML_NAMED_ELEMENT(VoteServiceState)
    QML_UNCREATABLE("仅用于公开服务状态枚举")

public:
    enum State {
        Stopped = VoteServiceState::Stopped,
        Starting = VoteServiceState::Starting,
        Listening = VoteServiceState::Listening,
        Connected = VoteServiceState::Connected,
        Stopping = VoteServiceState::Stopping,
        Error = VoteServiceState::Error
    };
    Q_ENUM(State)
};
```

它具有以下语义：

```text
Q_OBJECT
    → 按 QObject 类型生成元对象，不再作为小写值类型注册

QML_NAMED_ELEMENT(VoteServiceState)
    → QML 继续使用 VoteServiceState.Listening

QML_UNCREATABLE
    → QML 不能实例化该类型，只能读取类型级枚举

Q_ENUM(State)
    → 将适配枚举加入元对象并导出给 QML
```

该修复不会改变 Service、ViewModel、状态数值或 QML 调用方式。它只修正 presentation 层的注册载体。

## 15. 不推荐的替代方案

### 15.1 QML 直接比较数字

```qml
case 2:
```

这种写法缺少语义，枚举顺序调整后可能静默出错。

### 15.2 在 ViewModel 中重复定义状态枚举

这会让 Service 的领域状态和 ViewModel 的展示状态成为两套类型，需要额外转换，并使业务状态重新依附于 ViewModel。

### 15.3 让领域类型直接承担所有 QML 注册

这样会让 domain 层知道 `QML_ELEMENT` 等展示技术细节，并可能使同一元对象头文件被多个静态目标重复处理。除非项目明确接受该依赖方向，否则应由 presentation 适配。

### 15.4 为了枚举容器创建全局可变单例

枚举常量不需要可变全局对象。不要引入保存状态的 QML singleton 或 Service Locator 来解决类型注册问题。

## 16. 常见错误

1. **以为 `Q_OBJECT` 会自动注册 QML 类型**：还需要 QML 注册宏或运行时注册 API。
2. **以为 `Q_PROPERTY` 会生成成员和 Setter**：它只登记元数据。
3. **修改成员变量后漏发 `NOTIFY`**：QML Binding 不会自动刷新。
4. **值没有变化仍反复发通知**：会造成无意义刷新和难以验证的通知次数。
5. **把 signal 当作状态存储**：signal 是通知，不保存当前值。
6. **在 `Q_GADGET` 中声明 signals**：它不是 QObject，不能提供完整 signal/slot 能力。
7. **在没有元对象载体的类中单独使用 `Q_ENUM`**：枚举无处登记。
8. **把大写 QML 对象名注册成 gadget 值类型**：会触发值类型命名校验。
9. **同一 `Q_GADGET` 头文件在多个静态目标中重复 AUTOMOC**：可能产生重复 `staticMetaObject`。
10. **把 `QML_UNCREATABLE` 理解为不可见**：它仍然对 QML 可见，只是不能由 QML 创建实例。

## 17. 项目实践准则

- Service、ViewModel、Qt Model 等有身份、有生命周期并需要信号的类型使用 `Q_OBJECT`。
- 领域枚举和值对象在确实需要 Qt 元数据时使用 `Q_GADGET`。
- 只有需要反射或导出的枚举才增加 `Q_ENUM`。
- 当前状态通过成员变量和 `Q_PROPERTY + NOTIFY` 暴露；瞬时事件使用普通 signal。
- QML 注册宏放在 presentation 边界，避免领域层依赖 UI 技术。
- QML 只使用具名枚举，不比较魔法数字。
- 静态 QML 模块必须保留 Qt 自动生成的资源初始化目标。
- 修改类型注册方式后，应同时运行 C++ 单元测试、QML 模块加载测试和麒麟 GUI 启动测试；仅通过无界面测试不足以覆盖平台运行时注册差异。

## 18. 速查结论

```text
需要 signal、Q_PROPERTY 通知、Q_INVOKABLE 或 QObject 生命周期
    → QObject + Q_OBJECT

只是普通值类型或枚举容器，但需要 Qt 元数据
    → Q_GADGET

枚举需要名称和值反射
    → 在 Q_OBJECT/Q_GADGET 中增加 Q_ENUM

C++ 类型需要被 QML 模块认识
    → 再增加 QML_ELEMENT/QML_NAMED_ELEMENT

QML 可以看见类型和枚举，但不能创建实例
    → QML_UNCREATABLE
```
