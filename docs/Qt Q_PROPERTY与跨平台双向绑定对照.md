# Qt Q_PROPERTY 与跨平台双向绑定对照

## 1. 文档目标

本文面向熟悉 WPF 或 Android MVVM、正在学习 Qt Quick/QML 的开发者，系统说明：

- `Q_PROPERTY` 的读取、写入、通知、重置与可绑定属性如何协作；
- QML 的 `property`、`alias`、`required`、`default`、`readonly` 如何定义组件接口；
- `NOTIFY` 后面的名称如何与 `signals` 中的信号关联；
- C++ 状态如何驱动 QML Binding 自动刷新；
- QML 如何把用户输入写回 C++；
- Qt、WPF、Android XML Data Binding 和 Jetpack Compose 如何表达同类数据流。

本文以表决项目的 `VoteServerViewModel` 为主线，新增的独立代码片段是教学示例，不表示本仓库包含对应实现。示例以 Qt 6.8 为基线，`BINDABLE` 属于 Qt 6 的能力。不同框架之间只是职责近似，不代表类型、线程、生命周期或运行机制完全相同。

QML 组件自定义 signal、用户意图和分层事件流参见：[Qt QML Signal 与分层事件流](Qt%20QML%20Signal与分层事件流.md)。

界面操作如何调用 C++ 方法，以及命令可用性与 WPF/Android 的对应关系，参见：[Qt Q_INVOKABLE 与跨平台命令调用对照](Qt%20Q_INVOKABLE与跨平台命令调用对照.md)。

`moc`、`Q_OBJECT`、`Q_GADGET`、`Q_ENUM` 和 QML 类型注册的基础原理参见：[Qt 元对象系统与 QML 类型注册](Qt元对象系统与QML类型注册.md)。

官方参考：

- [Qt 6.8 Property System](https://doc.qt.io/qt-6.8/properties.html)
- [Qt 6.8：向 QML 暴露 C++ 属性](https://doc.qt.io/qt-6.8/qtqml-cppintegration-exposecppattributes.html)
- [Qt 6.8 QML Object Attributes](https://doc.qt.io/qt-6.8/qtqml-syntax-objectattributes.html)
- [Qt 6.8 Property Binding](https://doc.qt.io/qt-6.8/qtqml-syntax-propertybinding.html)
- [Qt 6.8 QObjectBindableProperty](https://doc.qt.io/qt-6.8/qobjectbindableproperty.html)
- [WPF Data Binding Overview](https://learn.microsoft.com/dotnet/desktop/wpf/data/)
- [Android Two-way Data Binding](https://developer.android.com/topic/libraries/data-binding/two-way)
- [Android LiveData 与 Data Binding](https://developer.android.com/topic/libraries/data-binding/architecture)
- [Jetpack Compose State Hoisting](https://developer.android.com/develop/ui/compose/state-hoisting)
- [Jetpack Compose State](https://developer.android.com/develop/ui/compose/state)

## 2. Q_PROPERTY 不是成员变量

下面的声明只是把一个属性加入 Qt 元对象系统：

```cpp
Q_PROPERTY(bool running READ running NOTIFY runningChanged)
```

对于上面这种显式指定 `READ` 和 `NOTIFY` 名称的声明，它不会自动生成：

- `m_running` 成员变量；
- `running()` Getter；
- `runningChanged()` 信号；
- 修改属性或发送通知的代码。

这不意味着所有 `Q_PROPERTY` 写法都必须手写 Getter/Setter：`MEMBER` 可以直接关联已有成员变量；Qt 6 的 `BINDABLE` 配合 `READ default` / `WRITE default` 可以生成元对象读写入口。它们仍需要实际存储与相应实现，详见第 4 章。

完整实现仍然需要显式编写：

```cpp
class VoteServerViewModel final : public QObject
{
    Q_OBJECT
    Q_PROPERTY(bool running READ running NOTIFY runningChanged)

public:
    bool running() const
    {
        return m_running;
    }

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

可以把宏拆成以下元数据理解：

```text
属性名       running
属性类型     bool
读取入口     running()
写入入口     无，因此对 QML 只读
变更通知     runningChanged()
实际存储     m_running，由类自行管理
```

## 3. NOTIFY 如何与 signals 串起来

### 3.1 NOTIFY 指向类中已经存在的信号

```cpp
Q_PROPERTY(bool running READ running NOTIFY runningChanged)

signals:
    void runningChanged();
```

`NOTIFY runningChanged` 中的 `runningChanged` 必须能解析到该类已有的信号。它不一定必须使用“属性名 + `Changed`”，但这是 Qt 官方推荐且最容易理解的命名方式。

例如下面的声明在语法上也可以成立：

```cpp
Q_PROPERTY(bool running READ running NOTIFY stateUpdated)

signals:
    void stateUpdated();
```

不过 QML 生成的属性变化处理器仍以属性名为准，即 `onRunningChanged`。为了避免信号名、属性名和 QML 处理器名称不一致，通常应使用 `runningChanged`。

### 3.2 moc 在编译期记录关联关系

包含 `Q_OBJECT` 的类会由 Meta-Object Compiler（`moc`）处理。生成的元对象信息会记录：

```text
running 属性
  ├── 类型：bool
  ├── READ：running()
  ├── WRITE：无
  └── NOTIFY：runningChanged()
```

QML 引擎拿到一个 `VoteServerViewModel` 对象后，可以通过 `QMetaObject` 查询这些信息，不需要在 QML 中手动连接 `runningChanged()`。

### 3.3 QML Binding 的完整通知链路

QML 页面：

```qml
ServiceControlPanel {
    running: voteViewModel.running
}
```

组件内部：

```qml
RowLayout {
    id: root

    property bool running: false

    Button {
        text: qsTr("启动服务")
        enabled: !root.running
    }

    Button {
        text: qsTr("停止服务")
        enabled: root.running
    }
}
```

运行过程：

```text
VoteServerViewModel::setRunning(true)
    ↓
m_running 从 false 变成 true
    ↓
emit runningChanged()
    ↓
QML 引擎根据 Q_PROPERTY 元数据找到 running
    ↓
重新调用 VoteServerViewModel::running()
    ↓
重新计算 running: voteViewModel.running
    ↓
重新计算 enabled: !root.running 和 enabled: root.running
    ↓
启动按钮禁用，停止按钮启用
```

QML 不直接操作 C++ 成员变量，C++ 也不需要查找和操作按钮对象。

## 4. Q_PROPERTY 的访问入口与修饰项

### 4.1 READ：从 ViewModel 读取状态

```cpp
Q_PROPERTY(QString userName READ userName NOTIFY userNameChanged)
```

QML 读取：

```qml
Label {
    text: viewModel.userName
}
```

QML 引擎会调用：

```cpp
QString UserViewModel::userName() const;
```

### 4.2 WRITE：向 ViewModel 写入值

```cpp
Q_PROPERTY(
    QString userName
    READ userName
    WRITE setUserName
    NOTIFY userNameChanged
)
```

QML 赋值：

```qml
viewModel.userName = "张三"
```

等价于调用：

```cpp
viewModel.setUserName(QStringLiteral("张三"));
```

Setter 应保证只在值真正变化时发送通知：

```cpp
void UserViewModel::setUserName(const QString &userName)
{
    if (m_userName == userName) {
        return;
    }

    m_userName = userName;
    emit userNameChanged();
}
```

这可以避免无意义的 QML 绑定重算，也能截断可能出现的反馈循环。

### 4.3 NOTIFY：通知依赖项重新读取

`NOTIFY` 不负责保存状态，信号可以携带新值。它的主要职责是告诉依赖方：

```text
这个属性可能变化了，请通过 READ 重新读取。
```

通知信号常用无参数形式：

```cpp
void userNameChanged();
```

也可以携带一个与属性类型兼容的参数：

```cpp
void userNameChanged(const QString &userName);
```

对于 QML Binding，无参数形式通常已经足够，因为 QML 会通过 `READ` 获取最新值。

### 4.4 CONSTANT：对象生命周期中保持不变

当前项目的后端名称使用：

```cpp
Q_PROPERTY(QString backendName READ backendName CONSTANT)
```

它表示同一个对象实例存活期间，`backendName()` 每次都返回相同值。QML 不需要订阅变化通知。

`CONSTANT` 属性不能同时声明 `WRITE` 或 `NOTIFY`：

```text
CONSTANT
  = 同一对象实例每次读取都返回相同值

NOTIFY / WRITE
  = 属性可能变化
```

如果常量属性返回一个 QObject 指针，不变的是指针引用；被引用对象自身的可写属性仍可能变化，不能把 `CONSTANT` 理解为递归冻结对象。

### 4.5 MEMBER：直接关联已有成员变量

```cpp
// 教学示例，放在包含 Q_OBJECT 的 QObject 派生类中
Q_PROPERTY(QString title MEMBER m_title NOTIFY titleChanged)

signals:
    void titleChanged();

private:
    QString m_title;
```

`MEMBER` 不创建 `m_title`，只是让元对象系统直接读取和写入它，因此这里不需要另外声明 Getter/Setter。

必须区分两条修改路径：

```cpp
object->setProperty("title", QStringLiteral("新标题"));
// 对于上述 MEMBER 属性，经元对象写入且值变化时会自动发出 titleChanged。

m_title = QStringLiteral("新标题");
// 类内部直接改成员不会自动发出通知；需要自行比较新旧值并 emit。
```

QML 的 `object.title = "新标题"` 也走属性写入入口。上述自动通知适用于这种由元对象直接管理写入的 `MEMBER` 属性；显式 `WRITE` Setter 的普通属性仍由 Setter 负责通知。

`MEMBER` 可以额外搭配 `READ` 或 `WRITE` 中的一个以控制访问，但不能同时搭配两者。只有 `MEMBER`、没有 `WRITE` 并不代表只读，因为 `MEMBER` 本身就提供写入路径。需要校验、业务规则或统一修改入口时，通常优先选择显式 `READ + WRITE + NOTIFY`。

### 4.6 RESET：恢复上下文约定的默认状态

```cpp
// 教学示例，省略与第 4.2 节相同的 Getter、Setter、存储和信号
Q_PROPERTY(QString userName READ userName WRITE setUserName
           RESET resetUserName NOTIFY userNameChanged)

public:
    void resetUserName()
    {
        setUserName(QStringLiteral("访客"));
    }
```

`RESET` 指向返回 `void`、不带参数的函数。它恢复的值由实现决定，可以是应用默认值，也可以是继承或环境相关的状态；宏不会自动推导该值。

可以通过元对象触发重置：

```cpp
const auto meta = object->metaObject();
const int index = meta->indexOfProperty("userName");
if (index >= 0) {
    const QMetaProperty property = meta->property(index); // 需要 <QMetaProperty>
    if (property.isResettable()) {
        property.reset(object);
    }
}
```

QML 对带有 `RESET` 的 C++ 属性赋 `undefined` 也会请求重置：

```qml
viewModel.userName = undefined
```

`RESET` 不会把函数自动变成 QML 可调用方法；若要写 `viewModel.resetUserName()`，还需要公开为 slot 或 `Q_INVOKABLE`。重置导致值变化时仍应正确通知，上例通过既有 Setter 完成。

### 4.7 BINDABLE：Qt 6 的 C++ 属性依赖追踪

传统方案使用“普通成员变量 + Getter/Setter + 手动 `NOTIFY`”。Qt 6 也可以用可绑定存储，在 C++ 中建立依赖，并通过 `QBindable<T>` 暴露绑定接口：

```cpp
// 教学示例：Qt 6，完整类声明
#include <QObject>
#include <QProperty>

class BindableCounter : public QObject
{
    Q_OBJECT
    Q_PROPERTY(int count READ default WRITE default
               BINDABLE bindableCount NOTIFY countChanged)

public:
    using QObject::QObject;

    QBindable<int> bindableCount()
    {
        return QBindable<int>(&m_count);
    }

signals:
    void countChanged();

private:
    Q_OBJECT_BINDABLE_PROPERTY(BindableCounter, int, m_count,
                              &BindableCounter::countChanged)
};
```

这里有三层职责：

- `Q_OBJECT_BINDABLE_PROPERTY` 创建与 QObject 关联的可绑定存储，并把 `countChanged` 接入变化通知。
- `bindableCount()` 返回 `QBindable<int>`，允许设置、检查绑定或读写值。
- `READ default` / `WRITE default` 让 `moc` 生成使用该绑定接口的元对象读写入口，不会生成名为 `count()` / `setCount()` 的普通 C++ 方法。

调用方可以建立 C++ 单向依赖：

```cpp
QProperty<int> source { 2 }; // 需要 <QProperty>
BindableCounter counter;
counter.bindableCount().setBinding([&source] {
    return source.value() * 2;
});
source = 3; // counter 的 count 变为 6，并触发 countChanged
```

`source` 必须在依赖它的绑定存活期间有效。直接给可绑定属性设置静态值通常会移除原绑定；`BINDABLE` 也不会自动建立两个独立属性之间的双向同步。

特别注意：`WRITE default` 生成的写入入口不会自行 `emit NOTIFY`。上例的通知来自可绑定存储上的变化回调，不能仅写 `NOTIFY countChanged` 就假设信号已接入。

QML 可以跟踪 `BINDABLE` 属性的变化；纯可绑定属性不一定需要 `NOTIFY`。上例同时保留通知信号，方便传统信号槽订阅。本文其余 `READ + NOTIFY` 示例仍是有效的常规方案，无需为此全面改写 ViewModel。

### 4.8 FINAL：禁止派生类型覆盖属性

```cpp
Q_PROPERTY(bool running READ running NOTIFY runningChanged FINAL)
```

`FINAL` 声明这个属性不应被派生类型覆盖，QML 派生类型不能重新声明同名属性，也有利于属性访问优化。它不表示值固定，也不表示只读；带 `WRITE` 的 `FINAL` 属性仍可写。

它不是 C++ 的类或虚函数 `final`：`moc` 不负责强制禁止所有 C++ 派生声明，开发者仍需遵守这个契约。

### 4.9 REQUIRED：QML 创建时必须提供属性

```cpp
// 教学示例，Getter/Setter/信号由类实现，类已注册为 QML 类型
Q_PROPERTY(QString userName READ userName WRITE setUserName
           NOTIFY userNameChanged REQUIRED)
```

QML 使用该类型时必须在创建过程中设置属性：

```qml
UserViewModel {
    userName: "张三"
}
```

遗漏必填属性会使 QML 实例创建失败。`moc` 不会强制普通 C++ 构造代码填写它，也不会替你检查字符串是否为空或内容是否合法。C++ 成员已有初始值不能代替 QML 调用方的必填赋值。

它对应 QML 的 `required property`（第 8.3 节），用于初始化契约；创建完成后是否可以继续修改，由属性的写入口决定。

### 4.10 其他元数据修饰项速查

| 修饰项 | 声明示例片段 | 用途及边界 |
| --- | --- | --- |
| `REVISION` | `REVISION(1, 2)` | 属性及通知信号的 API 修订信息；结合 QML 类型/模块版本注册使用，不是属性值的版本计数 |
| `DESIGNABLE` | `DESIGNABLE false` | 是否在设计工具的属性编辑器中显示，默认 `true`；不等于禁止运行时读写 |
| `SCRIPTABLE` | `SCRIPTABLE false` | 是否供脚本引擎访问，默认 `true`；不是对象的安全隔离机制 |
| `STORED` | `STORED false` | 表示状态保存工具是否应把它作为独立值存储，默认 `true`；不会自动保存到文件或数据库 |
| `USER` | `USER true` | 标记该类主要的用户可编辑属性，默认 `false`；常供 Widgets 的 delegate 使用，不会自动建立双向绑定 |

QML 的 `alias`、`default`、`readonly` 不能原样填入 `Q_PROPERTY`。C++ 默认属性使用 `Q_CLASSINFO("DefaultProperty", "content")` 指定，关联的 `content` 属性仍需另行实现；常规 C++ 只读属性使用 `READ` 且不提供 `WRITE` 或可写 `MEMBER`。

## 5. 一个信号能否通知多个属性

当前项目中：

```cpp
Q_PROPERTY(ServiceState serviceState
           READ serviceState
           NOTIFY serviceStateChanged)

Q_PROPERTY(QString statusText
           READ statusText
           NOTIFY serviceStateChanged)
```

`serviceState` 和 `statusText` 共用 `serviceStateChanged()`，因为它们总是在同一个操作中更新：

```cpp
m_serviceState = state;
m_statusText = text;
emit serviceStateChanged();
```

收到该信号后，依赖这两个属性的绑定都会重新读取：

```text
serviceStateChanged()
  ├── 重新读取 serviceState()
  └── 重新读取 statusText()
```

Qt 允许多个低频、总是一起变化的属性共用一个通知信号，但需要权衡：

- 优点：减少信号数量，原子地通知一组关联展示状态；
- 缺点：即使某个属性没有变化，依赖它的绑定也可能重新计算；
- 建议：变化时机不同或更新频繁时，拆成独立的 `statusTextChanged()`；
- 建议：对外 API 更重视语义清晰时，也优先使用一属性一信号。

## 6. 普通 Signal、NOTIFY 与状态建模

### 6.1 底层机制相同，语义职责不同

当前 `IVoteServer` 定义了三个信号：

```cpp
class IVoteServer : public QObject
{
    Q_OBJECT

public:
    virtual bool start(const QString &ip, quint16 port) = 0;
    virtual void stop() = 0;

signals:
    void controllerConnected(const QString &ip);
    void controllerDisconnected(const QString &ip);
    void errorOccurred(const QString &context,
                       const QString &message);
};
```

它们和下面的属性通知信号使用同一套 Qt 信号槽机制：

```cpp
Q_PROPERTY(bool running READ running NOTIFY runningChanged)

signals:
    void runningChanged();
```

共同点包括：

- 类都需要 `Q_OBJECT`；
- 都由 `moc` 生成元对象信息；
- 都通过 `emit` 发送；
- 都能使用 `connect()` 订阅；
- 都可以选择直接连接或队列连接；
- 都可以携带参数。

区别不在 signal 的语言机制，而在元对象关联和业务语义：

```text
普通 signal
  = 某个瞬时事件刚刚发生
  = 不属于任何 Q_PROPERTY
  = 收到后由订阅者决定如何处理

NOTIFY signal
  = 某个可读取属性可能发生变化
  = 通过 Q_PROPERTY 与 READ 属性关联
  = 收到后 QML 重新读取属性并计算绑定
```

`runningChanged()` 被 `Q_PROPERTY` 的 `NOTIFY` 引用，因此 Qt 知道它是 `running` 的属性通知。`controllerConnected()` 没有被任何 `Q_PROPERTY` 引用，所以它只是普通事件信号。

### 6.2 瞬时事件与当前状态

`controllerConnected(ip)` 表达：

```text
控制器刚刚发生了一次连接事件，事件参数是 ip。
```

它不表达：

```text
控制器现在是否仍然连接。
```

signal 自身不保存当前值或历史。如果订阅者在信号发出之后才执行 `connect()`，它不会自动收到之前的连接事件。

属性则表示可以随时重新读取的当前状态：

```cpp
Q_PROPERTY(ServiceState serviceState
           READ serviceState
           NOTIFY serviceStateChanged)
```

即使界面没有见到最初的连接事件，也可以调用 `serviceState()` 得知当前状态。

两类信息可以这样区分：

| 问题 | 类型 | 推荐建模 |
| --- | --- | --- |
| 控制器刚才是否连接了一次 | 瞬时事件 | 普通 signal |
| 控制器现在是否连接 | 当前状态 | `Q_PROPERTY + NOTIFY` |
| 刚才发生了什么错误 | 瞬时事件 | 普通 signal |
| 页面当前显示什么错误 | 当前状态 | `Q_PROPERTY + NOTIFY` |
| 发生过哪些错误 | 事件历史 | Model 或集合属性 |

### 6.3 门铃与门状态

可以用门铃理解普通事件信号：

```cpp
signals:
    void doorbellRang();
```

门铃响过以后，新来的订阅者无法通过门铃知道刚才是否有人按过。它只能收到未来再次发生的事件。

门的开关状态则适合属性：

```cpp
Q_PROPERTY(bool doorOpen
           READ doorOpen
           NOTIFY doorOpenChanged)
```

任何时候都可以调用 `doorOpen()` 查询当前状态。`doorOpenChanged()` 只负责提醒：

```text
门的状态已经变化，请重新读取 doorOpen。
```

对应当前项目：

```text
controllerConnected(ip)
  = 门铃事件：控制器刚刚连接

serviceState == Connected
  = 门状态：当前界面状态是已连接

serviceStateChanged()
  = 状态通知：请重新读取 serviceState 和 statusText
```

### 6.4 IVoteServer 事件如何转换为 ViewModel 状态

`VoteServerViewModel` 在构造时订阅底层事件：

```cpp
connect(m_voteServer.get(),
        &IVoteServer::controllerConnected,
        this,
        [this](const QString &ip) {
            setServiceState(ServiceState::Connected, ip);
            appendLog(QStringLiteral("控制器已连接：%1").arg(ip));
        });

connect(m_voteServer.get(),
        &IVoteServer::controllerDisconnected,
        this,
        [this](const QString &ip) {
            setServiceState(ServiceState::Disconnected, ip);
            appendLog(QStringLiteral("控制器已断开：%1").arg(ip));
        });

connect(m_voteServer.get(),
        &IVoteServer::errorOccurred,
        this,
        [this](const QString &context, const QString &message) {
            setServiceState(ServiceState::Error, message);
            appendLog(QStringLiteral("错误 [%1]：%2")
                          .arg(context, message));
        });
```

这段代码把一次底层事件转换成两类可展示信息：

```text
底层事件
controllerConnected(ip)
    ├── 当前状态：serviceState/statusText
    └── 事件历史：logEntries
```

随后 `setServiceState()` 更新状态并发送属性通知：

```cpp
m_serviceState = state;
m_statusText = text;
emit serviceStateChanged();
```

完整链路是：

```text
真实 SDK 或 MockVoteServer
    ↓
emit controllerConnected(ip)
    ↓
IVoteServer 普通 signal 表达瞬时事件
    ↓
VoteServerViewModel 收到并解释事件
    ├── 更新 serviceState/statusText
    └── 追加 logEntries
    ↓
emit serviceStateChanged()/logEntriesChanged()
    ↓
QML 重新读取属性并刷新页面
```

这里同时存在两种信号是必要的：

- `IVoteServer` 只负责上报后端发生的事实，不承担页面状态；
- ViewModel 把事实归纳为当前状态和历史记录；
- QML 只绑定最终状态，不直接维护 SDK 事件状态机。

### 6.5 为什么不让 QML 直接监听 IVoteServer

技术上可以把 `IVoteServer` 暴露给 QML，再用 `Connections` 监听：

```qml
Connections {
    target: voteServer

    function onControllerConnected(ip) {
        // 页面自行更新状态
    }
}
```

但这样会导致：

- View 直接依赖后端接口；
- 页面必须自行维护状态机和日志；
- 页面重建后无法仅凭过去事件恢复当前状态；
- 多个页面容易重复解释同一事件；
- Mock、真实 SDK 和界面职责混合；
- 业务事件转换难以脱离 QML 测试。

当前架构保持以下边界：

```text
IVoteServer
  提供后端操作并发出基础设施事件

VoteServerViewModel
  将事件转换成界面可读取状态

QML
  显示状态并发送用户意图
```

### 6.6 与 WPF 和 Android 的对应关系

WPF 中通常使用不同机制表达这两类信息：

```csharp
// 当前状态及其变化通知
public bool Running { get; private set; }
public event PropertyChangedEventHandler? PropertyChanged;

// 瞬时事件
public event EventHandler<ControllerConnectedEventArgs>?
    ControllerConnected;
```

近似对应：

```text
Qt Q_PROPERTY + NOTIFY
  ≈ WPF 属性 + INotifyPropertyChanged

Qt 普通 signal
  ≈ C# 普通 event
```

现代 Android 通常进一步区分：

```text
StateFlow / LiveData
  保存并暴露当前状态

SharedFlow / Channel
  传递瞬时事件
```

近似对应：

```text
Qt Q_PROPERTY + NOTIFY
  ≈ StateFlow / LiveData

Qt 普通 signal
  ≈ SharedFlow / Channel
```

这些只是职责对照：

- 普通 Qt signal 不保存历史值；
- `StateFlow` 保存当前值，`SharedFlow` 是否重放取决于配置；
- `LiveData` 保存最近值并具有 Android 生命周期感知；
- `Channel` 具有缓冲、关闭和消费语义；
- QObject、WPF 对象和 Android ViewModel 的生命周期并不相同。

不要因为它们承担相似职责，就假设线程、重放、背压或生命周期行为完全一致。

### 6.7 如何选择

判断一个信息应该使用属性还是普通 signal，可以先问：

> 新订阅者加入后，是否必须立即知道当前值？

如果答案是“是”，优先使用状态属性：

```cpp
Q_PROPERTY(bool connected
           READ connected
           NOTIFY connectedChanged)
```

如果答案是“不需要，只关心事件发生”，使用普通 signal：

```cpp
signals:
    void controllerConnected(const QString &ip);
```

如果既要当前状态又要历史，应分别建模：

```text
当前状态     Q_PROPERTY + NOTIFY
新事件       普通 signal
事件历史     QAbstractListModel 或集合属性
```

不要为了避免定义属性而让页面自行记忆事件，也不要为了保留每次事件而反复覆盖一个“当前值”属性。

## 7. Qt/QML 的单向绑定与双向写回

### 7.1 READ + NOTIFY 是响应式单向绑定

```cpp
Q_PROPERTY(bool running READ running NOTIFY runningChanged)
```

```qml
Button {
    enabled: !viewModel.running
}
```

数据方向是：

```text
ViewModel.running
    ↓
View.enabled
```

因为没有 `WRITE`，QML 不能执行：

```qml
viewModel.running = true // 不允许
```

### 7.2 WRITE 只提供写入口，不自动创建双向绑定

即使 C++ 属性包含 `WRITE`：

```cpp
Q_PROPERTY(QString userName
           READ userName
           WRITE setUserName
           NOTIFY userNameChanged)
```

下面仍然只是从 ViewModel 到 View 的绑定：

```qml
TextField {
    text: viewModel.userName
}
```

QML 通常通过输入事件显式写回：

```qml
TextField {
    text: viewModel.userName
    onTextEdited: viewModel.userName = text
}
```

完整双向流程：

```text
ViewModel → View
setUserName() 或其他业务更新
    ↓
emit userNameChanged()
    ↓
TextField.text 重新读取 viewModel.userName

View → ViewModel
用户编辑 TextField
    ↓
onTextEdited
    ↓
viewModel.userName = text
    ↓
调用 setUserName(text)
    ↓
更新成员变量并发送 userNameChanged()
```

### 7.3 根据业务选择写回时机

每次用户编辑都写回：

```qml
TextField {
    text: viewModel.searchKeyword
    onTextEdited: viewModel.searchKeyword = text
}
```

完成编辑或失焦后写回：

```qml
TextField {
    text: viewModel.userName
    onEditingFinished: viewModel.userName = text
}
```

显式提交时写回：

```qml
TextField {
    id: portInput
    text: viewModel.portText
}

Button {
    text: qsTr("保存")
    onClicked: viewModel.applyPort(portInput.text)
}
```

不要默认使用 `onTextChanged` 写回，因为代码更新 `text` 也可能触发它，产生多余 Setter 调用或反馈环。表达“用户进行了编辑”时优先使用 `onTextEdited`。

### 7.4 系统状态不应为了双向绑定而增加 WRITE

当前项目的这些属性应保持只读：

```cpp
running
serviceState
statusText
logEntries
```

错误做法：

```qml
voteViewModel.running = true
```

这只会修改显示状态，并没有真正启动 SDK。正确方式是发送用户意图：

```qml
voteViewModel.startServer()
```

然后由 ViewModel：

```text
调用 IVoteServer::start()
    ↓
确认启动成功
    ↓
setRunning(true)
    ↓
emit runningChanged()
```

适合 `WRITE` 的通常是可以直接编辑且 Setter 能完整表达修改语义的值，例如用户名、搜索词和表单草稿。涉及校验、异步操作、设备状态机或失败处理时，优先使用明确命令。

### 7.5 命令式赋值会覆盖已有绑定

```qml
// 教学示例：BindingDemo.qml
import QtQuick

Item {
    id: root
    property int source: 2
    property int doubled: root.source * 2

    Component.onCompleted: {
        root.source = 3       // doubled 随绑定变为 6
        root.doubled = 100    // 对绑定目标赋静态值，移除原绑定
        root.source = 4       // doubled 仍为 100

        root.doubled = Qt.binding(function() {
            return root.source * 2
        })                   // 重新建立绑定，doubled 变为 8
        root.source = 5       // doubled 变为 10
    }
}
```

`doubled: root.source * 2` 在声明处建立依赖关系；JavaScript 中的 `doubled = root.source * 2` 只计算并写入当时的值，必须使用 `Qt.binding()` 才能在命令式代码中建立持续绑定。恢复的是这次指定的表达式，并非自动找回历史绑定。

因此，界面写回应修改状态源或调用命令，避免随手给本应由绑定计算的目标属性赋值。通过 alias 给目标赋静态值，也会影响目标原有的绑定。

## 8. QML 属性声明与组件接口

本章是 Qt 6.8 的独立教学示例。C++ `Q_PROPERTY` 把属性接入元对象系统，QML 的属性声明则定义组件自身的接口或状态；两者可以参与同一条绑定链路，但关键字不能直接互换。

### 8.1 property：存储属性与响应式表达式

```qml
// PropertyDemo.qml
import QtQuick

Item {
    id: root
    property string userName: "张三"        // 可写的普通属性
    property int count: 0                   // 初始静态值
    property string greeting: "你好，" + root.userName // 单向绑定

    onUserNameChanged: console.log(root.greeting)
}
```

普通 QML 属性拥有自己的存储，声明时会自动具备变化通知和 `on<PropertyName>Changed` 处理器，不需要像传统 C++ 属性一样手写信号与 `emit`。绑定表达式的依赖发生变化时，引擎重新计算结果。

`greeting` 即使有绑定，仍是可写属性；命令式写入会覆盖绑定（第 7.5 节）。在属性变化处理器中也不要假设其他相关绑定已经按固定顺序全部更新。

### 8.2 alias：把内部属性作为组件接口

```qml
// AliasInput.qml
import QtQuick
import QtQuick.Controls

Item {
    id: root
    width: 240
    height: input.implicitHeight
    property alias text: input.text
    property alias editor: input

    TextField {
        id: input
        anchors.fill: parent
    }
}
```

调用方可以写：

```qml
AliasInput {
    id: nameInput
    text: "张三"
    Component.onCompleted: {
        nameInput.text = "李四" // 直接修改内部 input.text
        console.log(nameInput.editor.text) // 读取内部控件的属性
    }
}
```

`alias` 不复制值、不创建第二份文本状态。`nameInput.text` 和内部 `input.text` 是同一属性的两个访问入口；内部文本变化也会反映到 alias，alias 变化可以通过 `onTextChanged` 观察。

它与下面的普通属性不同：

```qml
property string text: input.text
```

这只建立 `input.text → root.text` 的单向绑定。给 `root.text` 赋值会覆盖这个绑定，不会写入 `input.text`。

alias 的引用有明确限制：

- 必须在声明时指定目标，不能写任意 JavaScript 表达式或函数调用。
- 目标必须通过当前组件作用域内的 `id` 引用，不能直接引用调用方上下文里的 ViewModel。
- 不能引用 attached property；目标可以是对象、对象属性或允许的值类型子属性，例如 `rect.border.color`，不能无限串接路径，例如 `item.rect.border.color`。
- 不声明显式类型，类型来自目标的声明类型；对象 alias 不保证暴露目标内联新增的全部属性。优先暴露具体属性，必要时把内部对象提取为独立命名组件。

对象 alias 是对象引用入口，不是复制对象；是否能写入属性 alias 取决于目标的可写性。暴露整个控件会增加调用方对内部实现的依赖，通常只暴露必要属性和用户意图信号。

即使调用方写 `text: viewModel.userName`，alias 也不会自动把编辑结果写回 ViewModel。它只是把绑定放到了内部 `input.text` 上；跨对象的反向写回仍需要用户编辑事件或命令，第 8.7 节给出完整例子。

### 8.3 required：创建实例时的必填输入

```qml
// RequiredLabel.qml
import QtQuick

Text {
    required property string userName
    text: "用户：" + userName
}
```

调用方在创建时提供值或绑定：

```qml
RequiredLabel {
    userName: "张三"
}
```

遗漏时不能成功创建：

```qml
RequiredLabel {} // 错误示例：缺少 required 属性 userName
```

静态加载会报告缺少必填属性并导致相应组件加载失败；动态 `createObject()` 创建失败时返回 `null`，不要继续使用该对象。

```qml
// 教学片段：在已加载组件的调用方中执行
const component = Qt.createComponent("RequiredLabel.qml")
if (component.status === Component.Ready) {
    const label = component.createObject(parentItem, { userName: "张三" })
    if (label === null)
        console.warn(component.errorString())
}
```

异步加载需要等待 `statusChanged`；不能在 `Component.onCompleted` 中才补填，因为必填检查属于创建过程。C++ 创建入口可用 `QQmlComponent::createWithInitialProperties()` 或引擎的 `setInitialProperties()` 提供初值。

也可以把已有属性标记为必填：

```qml
// RequiredRectangle.qml
import QtQuick

Rectangle {
    required color
}
```

调用时写 `RequiredRectangle { color: "steelblue" }`。不要在同一必填属性声明处写 `required property string userName: "访客"`，这是错误示例；应由实例调用方提供值。

`required` 只要求初始化，不验证业务内容：空字符串仍是一个已提供的值；创建之后普通必填属性仍可修改。

在 delegate 中，它还能明确声明需要哪些模型角色：

```qml
// RoleList.qml
import QtQuick

ListView {
    width: 240
    height: 120
    model: ListModel {
        ListElement { userName: "张三" }
        ListElement { userName: "李四" }
    }
    delegate: Text {
        required property string userName // 由同名模型角色初始化
        required property int index       // 声明需要的索引
        text: (index + 1) + ". " + userName
    }
}
```

使用 required 属性的 delegate 应显式声明需要的角色，以及需要时的 `index`、`model` 或 `modelData`，不能依赖所有角色都作为隐式上下文变量可见。

### 8.4 default：接收省略属性名的子对象

`default` 指定组件的默认内容入口，不是给属性设置默认值。`property int count: 10` 中的 `10` 才是初始值。

```qml
// ContentBox.qml
import QtQuick

Item {
    id: root
    width: 240
    height: 120
    default property alias content: contentItem.data

    // 显式放进根对象原有的 data，避免内部容器进入自己接收的 content。
    data: Item {
        id: contentItem
        anchors.fill: parent
    }
}
```

调用方可以省略 `content:`：

```qml
ContentBox {
    Text { text: "内容区域" }
    Rectangle { y: 30; width: 80; height: 20; color: "steelblue" }
}
```

这些对象进入 `contentItem.data`；其中的可视 Item 的视觉父对象是 `contentItem`。也可显式写 `content: [ Text { ... }, Rectangle { ... } ]`。

一个类型只有一个有效默认属性，可以用自定义默认属性替代继承的默认入口。`Item` 原有的默认属性是 `data`，并不是 `children`；`data` 能接收可视 Item 和非可视 QObject，`children` 只列出可视子项。

在同一个对象定义中，不要对同一个默认列表既使用隐式子对象又显式赋列表，以免元素顺序不确定。上例内部的根 `data` 和调用方的 `contentItem.data` 是两个不同列表。

只定义 `default property list<Item> content` 可以保存对象列表，但不会自动完成容器布局或视觉父对象设置；alias 到内部 Item 的 `data` 是常见封装方式。C++ 对应默认入口元数据使用 `Q_CLASSINFO("DefaultProperty", "content")`，不是 `Q_PROPERTY DEFAULT`。

### 8.5 readonly：禁止重新赋值，绑定结果仍可变化

```qml
// ReadonlyDemo.qml
import QtQuick

Item {
    id: root
    property int count: 0
    readonly property int doubled: root.count * 2
    readonly property string appTitle: "属性示例"
    readonly property QtObject settings: QtObject {
        property string theme: "light"
    }

    Component.onCompleted: {
        root.count = 3             // doubled 自动变为 6
        root.settings.theme = "dark" // 可以修改引用对象的可写属性
        // root.doubled = 100      // 错误示例：不能给只读属性赋值
        // root.settings = null   // 错误示例：不能替换只读对象引用
    }
}
```

`readonly` 属性必须在声明处提供静态值或绑定。初始化后不能替换该值或绑定表达式，但绑定依赖变化时仍会产生新结果与属性变化通知。

| 声明方式 | 调用方能否重新赋值 | 读取结果能否随依赖/业务变化 |
| --- | --- | --- |
| QML `property int doubled: count * 2` | 能，赋静态值会覆盖绑定 | 能 |
| QML `readonly property int doubled: count * 2` | 不能 | 能，由原绑定重新计算 |
| C++ 常规 `READ + NOTIFY`，没有写入口 | QML 不能写 | 能，由 C++ 内部更新并通知 |
| C++ `READ + CONSTANT` | QML 不能写 | 同一实例读取结果不变 |

因此 `readonly` 不等于 `CONSTANT`。它也不是 C++ 的“类内可写、类外只读”Setter 访问权限：QML 自己的 JavaScript 方法同样不能重新赋值给该只读属性。

对对象引用而言，只读限制的是引用替换，不会递归禁止对象内部属性修改；值类型子属性则涉及整体值写回，不能套用对象引用的行为。

### 8.6 组合规则与选择速查

Qt 6.8 的常用写法是 `[default] [required] [readonly] property 类型 名称`，alias 使用 `property alias 名称: 目标`；这些可选词不能任意叠加。

| 写法 | 适用场景或限制 |
| --- | --- |
| `property string title: "标题"` | 可写属性，可以设置静态初值或绑定 |
| `property alias text: input.text` | 暴露已有属性，不存储第二份值 |
| `required property string title` | 调用方在创建时提供必填输入，不能在声明处初始化 |
| `required color` | 把已有属性标记为必填，不重新声明类型 |
| `default property list<Item> content` | 定义默认子对象入口，布局和父子关系仍需组件处理 |
| `default property alias content: inner.data` | 把默认入口转发到内部容器 |
| `readonly property bool ready: count > 0` | 暴露由依赖推导的只读状态，声明处必须初始化 |
| `default readonly property ...` | 官方不支持的组合：只读属性不能作为默认属性；不要依赖工具在声明阶段拒绝它 |
| `required readonly property ...` | 错误组合：必填输入与只读初始化契约冲突 |
| `required property alias ...` | 错误组合：alias 不能这样声明为 required |

检查边界：本机 Qt 6.8.3 的引擎和 `qmllint` 接受了单独的 `default readonly` 声明，因此“工具未报错”不能代替官方使用约束。其余上述错误组合，以及 required 声明处带初值、readonly 缺少初值，在创建验证中会报错。

这些关键字分别控制存储入口、引用关系、初始化要求、子对象接收位置和写入权限。它们没有 WPF `Mode=TwoWay` 或 Android `@={...}` 的通用同步含义。

### 8.7 组合示例：必填标题、文本 alias、只读状态与默认内容

```qml
// UserNameEditor.qml：教学组件
import QtQuick
import QtQuick.Controls

Item {
    id: root
    required property string title
    property alias text: input.text
    readonly property bool hasText: input.text.trim().length > 0
    default property alias content: extraContent.data
    signal textEdited(string value)

    implicitWidth: 240
    implicitHeight: column.implicitHeight

    data: Column {
        id: column
        width: root.width
        spacing: 8

        Label { text: root.title }
        TextField {
            id: input
            width: parent.width
            onTextEdited: root.textEdited(text)
        }
        Item {
            id: extraContent
            width: parent.width
            implicitHeight: childrenRect.height
            height: implicitHeight
        }
    }
}
```

调用方只绑定值，并在编辑事件中写回状态源：

```qml
// EditorPage.qml：与 UserNameEditor.qml 放在同一目录
import QtQuick
import QtQuick.Controls

Item {
    id: page
    width: 320
    height: 200
    property string userName: "张三" // 教学用状态源，业务中可替换为 ViewModel

    UserNameEditor {
        id: editor
        title: "用户名"            // 满足 required
        text: page.userName        // 经 alias 绑定到内部 TextField.text
        onTextEdited: function(value) {
            page.userName = value  // 显式写回状态源
        }

        Label { text: editor.hasText ? "已填写" : "请输入用户名" }
        // 省略 content:，Label 会进入 extraContent.data。
    }
}
```

数据流仍是：状态源 → alias 目标 → 控件显示；用户编辑 → 组件信号 → 调用方写回状态源。`hasText` 是派生状态，默认内容是扩展展示入口，二者不参与反向赋值。

## 9. WPF 对照

### 9.1 INotifyPropertyChanged

```csharp
public sealed class VoteServerViewModel : INotifyPropertyChanged
{
    private bool _running;

    public bool Running
    {
        get => _running;
        private set
        {
            if (_running == value)
                return;

            _running = value;
            PropertyChanged?.Invoke(
                this,
                new PropertyChangedEventArgs(nameof(Running)));
        }
    }

    public event PropertyChangedEventHandler? PropertyChanged;
}
```

对应关系：

```text
Qt READ running            ≈ C# Getter
Qt WRITE setRunning        ≈ C# Setter
Qt runningChanged()        ≈ PropertyChanged(nameof(Running))
QML Binding                ≈ XAML Binding
```

WPF 通常使用一个通用的 `PropertyChanged` 事件加属性名，Qt 通常为每个属性声明独立、强类型的通知信号。

### 9.2 OneWay 与 TwoWay

单向：

```xml
<TextBlock Text="{Binding UserName, Mode=OneWay}" />
```

双向：

```xml
<TextBox Text="{Binding UserName,
                        Mode=TwoWay,
                        UpdateSourceTrigger=PropertyChanged}" />
```

WPF 的 `Mode=TwoWay` 会在目标控件变化时自动调用源属性 Setter。Qt/QML 通常用“绑定 + 事件写回”显式表达两条方向：

```qml
TextField {
    text: viewModel.userName
    onTextEdited: viewModel.userName = text
}
```

### 9.3 UpdateSourceTrigger 与 QML 写回时机

| WPF | QML 近似行为 |
| --- | --- |
| `PropertyChanged` | `onTextEdited` 每次编辑写回 |
| `LostFocus` | `onEditingFinished` 写回 |
| `Explicit` | 保存按钮调用命令写回 |

### 9.4 DependencyProperty

WPF 自定义控件常使用 DependencyProperty：

```csharp
public static readonly DependencyProperty RunningProperty =
    DependencyProperty.Register(
        nameof(Running),
        typeof(bool),
        typeof(ServiceControl),
        new PropertyMetadata(false));
```

它更接近 QML 控件自身的声明式属性系统。业务 ViewModel 一般仍使用普通 CLR Property 加 `INotifyPropertyChanged`，不要把 ViewModel 属性与控件 DependencyProperty 混为一谈。

## 10. Android XML Data Binding 对照

### 10.1 @Bindable 与通知

```kotlin
class UserViewModel : BaseObservable() {
    @get:Bindable
    var userName: String = ""
        set(value) {
            if (field == value) return
            field = value
            notifyPropertyChanged(BR.userName)
        }
}
```

单向绑定：

```xml
<TextView android:text="@{viewModel.userName}" />
```

这里的 `@Bindable + BR + notifyPropertyChanged()` 在职责上接近 `Q_PROPERTY + NOTIFY`。

### 10.2 @={} 双向绑定

```xml
<EditText android:text="@={viewModel.userName}" />
```

区别是：

```text
@{...}    单向：ViewModel → View
@={...}   双向：ViewModel ⇄ View
```

Android Data Binding 会组合正向赋值和反向监听，自动把输入写回属性。Qt/QML 没有完全对应的通用 `TwoWay` 标记，通常显式写成：

```qml
TextField {
    text: viewModel.userName
    onTextEdited: viewModel.userName = text
}
```

### 10.3 MutableLiveData

```kotlin
class UserViewModel : ViewModel() {
    val userName = MutableLiveData("")
}
```

```xml
<EditText android:text="@={viewModel.userName}" />
```

`MutableLiveData` 同时具有当前值、可写入口和观察通知，因此在职责上接近包含 `READ + WRITE + NOTIFY` 的 Qt 属性。但 `LiveData` 具有 Android 生命周期感知语义，普通 `QObject` 和 Qt 信号本身没有完全相同的能力。

## 11. Jetpack Compose 对照

Compose 更推荐状态提升和单向数据流，而不是隐藏式双向绑定：

```kotlin
class UserViewModel : ViewModel() {
    private val _userName = MutableStateFlow("")
    val userName: StateFlow<String> = _userName.asStateFlow()

    fun updateUserName(value: String) {
        _userName.value = value
    }
}
```

```kotlin
@Composable
fun UserNameInput(viewModel: UserViewModel) {
    val userName by viewModel.userName.collectAsStateWithLifecycle()

    TextField(
        value = userName,
        onValueChange = viewModel::updateUserName
    )
}
```

数据流是：

```text
状态向下
ViewModel.userName
    ↓
TextField.value

事件向上
用户输入
    ↓
onValueChange
    ↓
viewModel.updateUserName()
    ↓
StateFlow 发出新值
    ↓
Compose 重组
```

它产生双向交互效果，但架构上仍然是单向数据流。这与较严格的 QML 写法非常接近：

```qml
TextField {
    text: viewModel.userName
    onTextEdited: viewModel.updateUserName(text)
}
```

相较于直接写 `viewModel.userName = text`，命令方法更适合包含校验、格式化或业务规则的修改。

## 12. 四套实现的统一对照

| 能力 | Qt/QML | WPF | Android XML Data Binding | Jetpack Compose |
| --- | --- | --- | --- | --- |
| 状态存储 | C++ 成员变量 | C# 字段 | 普通字段/`LiveData` | `StateFlow`/`State` |
| 对外读取 | `Q_PROPERTY READ` | Getter | Getter/`LiveData` | 只读 `StateFlow` |
| 对外写入 | `Q_PROPERTY WRITE` | Setter | Setter/`MutableLiveData` | ViewModel 方法 |
| 变化通知 | `NOTIFY` 信号 | `INotifyPropertyChanged` | `notifyPropertyChanged`/LiveData | Flow 发出新值 |
| 当前状态流 | `Q_PROPERTY + NOTIFY` | 属性 + `INotifyPropertyChanged` | `LiveData` | `StateFlow`/`State` |
| 瞬时事件 | 普通 signal | C# event | 事件包装/回调 | `SharedFlow`/`Channel` |
| 事件历史 | `QAbstractListModel`/集合属性 | `ObservableCollection<T>` | 集合/数据库 | 状态集合/数据层 |
| 单向绑定 | QML Binding | `Mode=OneWay` | `@{...}` | 状态参数/collect |
| 反向写回 | 输入事件赋值或命令 | Setter | Inverse Binding | `onValueChange` |
| 双向表达 | Binding + 显式写回 | `Mode=TwoWay` | `@={...}` | 状态向下、事件向上 |
| 写回时机 | 选择 QML 信号 | `UpdateSourceTrigger` | 属性监听器/适配器 | 事件回调位置 |
| 生命周期感知 | QObject 生命周期 | DataContext/控件生命周期 | `LifecycleOwner` | Composition + ViewModel |

统一理解：

```text
状态拥有者修改数据
    ↓
发出变化通知
    ↓
声明式界面更新依赖该状态的部分

用户产生输入或操作
    ↓
界面通过 Setter 或命令上报意图
    ↓
状态拥有者校验并更新单一事实来源
```

## 13. 常见错误

### 13.1 修改成员变量但漏发 NOTIFY

错误：

```cpp
m_running = true;
```

结果是 C++ 中的值已经变化，QML 仍显示旧状态。应统一通过 Setter：

```cpp
setRunning(true);
```

### 13.2 值没有变化也发送信号

错误：

```cpp
void setRunning(bool running)
{
    m_running = running;
    emit runningChanged();
}
```

正确：

```cpp
if (m_running == running) {
    return;
}
```

否则会造成不必要的绑定计算，复杂依赖下还可能引发反馈循环。

### 13.3 把 WRITE 当成自动 TwoWay Binding

```cpp
Q_PROPERTY(QString name READ name WRITE setName NOTIFY nameChanged)
```

只是让 QML 可以写入 `name`。要形成双向交互，还需要选择输入事件并显式写回。

### 13.4 使用 onTextChanged 无差别写回

程序刷新文本也可能触发 `textChanged`。如果目标是响应用户编辑，优先使用：

```qml
onTextEdited: viewModel.name = text
```

### 13.5 直接写业务状态

不要为了方便双向绑定，把 `running`、`connected` 或 `serviceState` 暴露为可写属性。它们是业务操作结果，不是用户表单输入。

### 13.6 所有属性共用一个通知信号

Qt 允许共享 `NOTIFY`，但大范围共享会使无关绑定一起重新计算。只有属性总是一起变化、使用频率较低且语义明确时才共享。

### 13.7 把 alias 当成两个状态源之间的双向绑定

`property alias text: input.text` 暴露的是同一目标属性。调用方的 `text: viewModel.userName` 仍是单向绑定，需要通过用户编辑信号显式写回 ViewModel；给 alias 静态赋值还可能覆盖目标绑定。

### 13.8 把 readonly 当成常量或类内可写属性

`readonly property int doubled: count * 2` 的结果可以随 `count` 更新，但组件内部 JavaScript 也不能执行 `doubled = 100`。只读 QObject 引用的内部可写属性仍可修改；`CONSTANT` 约束同一 C++ 实例的读取结果，二者不能等同。

### 13.9 把 default 当成默认值，把 required 当成校验

`default` 决定省略属性名的子对象进入哪里，初始值仍用 `:` 指定。`required` 要求创建时提供输入，不能用 `Component.onCompleted` 补填，也不保证输入非空或业务合法。

### 13.10 以为 MEMBER 或 WRITE default 总会自动通知

直接修改 `MEMBER` 对应的成员变量不会自动通知。`WRITE default` 依赖可绑定存储的通知回调，必须正确接入；声明 `NOTIFY` 名称本身不会完成接线。普通显式 Setter 的通知仍由实现负责。

## 14. 项目实践准则

当前表决项目统一采用以下原则：

1. 后端和服务状态使用 `READ + NOTIFY`，QML 只读。
2. 启动、停止和模拟事件通过 `Q_INVOKABLE` 命令表达用户意图。
3. Setter 先比较新旧值，再修改成员并发送通知。
4. 视觉属性由 QML 管理，不暴露 `statusColor` 等 View 属性。
5. 表单类数据可以使用 `READ + WRITE + NOTIFY`，但包含业务规则时优先使用命令方法。
6. 输入框使用 `onTextEdited`、`onEditingFinished` 或显式保存命令决定写回时机。
7. 共享 `NOTIFY` 必须保证语义相关且更新时机一致，否则拆分通知信号。
8. 当前状态使用 `Q_PROPERTY + NOTIFY`，保证新界面可以随时读取。
9. 连接、断开和错误等瞬时事实使用普通 signal，不把 signal 当成状态存储。
10. 需要展示事件历史时使用 `QAbstractListModel` 或集合属性，不依赖页面自行记忆事件。
11. QML 组件用 `required` 声明必填输入，用 `readonly` 暴露派生状态，用 `alias` 暴露必要内部属性，用 `default` 定义内容入口。
12. 组件优先转发用户编辑信号，由调用方决定写回时机；alias 不代替业务层的 Setter 或命令。
13. 只读、常量、必填和禁止覆盖各有不同职责，不用 `readonly`、`CONSTANT`、`REQUIRED`、`FINAL` 互相替代。
14. 避免命令式修改绑定目标；需要建立持续绑定时使用声明式表达式或 `Qt.binding()`。
15. 采用 `MEMBER` 或 `BINDABLE` 前先明确修改路径与通知来源；包含业务规则的属性仍优先使用显式方法。

可以用下面这组结论快速判断：

```text
READ + NOTIFY
  = ViewModel → View 的响应式单向绑定

READ + WRITE + NOTIFY
  = 属性具备双向数据流基础

QML Binding + 输入事件写回
  = 实际双向交互

系统状态 + 命令方法
  = 状态向下、意图向上，适合业务操作

普通 signal
  = 瞬时事件，只通知当前订阅者

Q_PROPERTY + NOTIFY
  = 可持续读取的当前状态

Model / 集合属性
  = 可以查询和展示的事件历史

QML alias
  = 已有属性或对象的另一个访问入口

QML required / C++ REQUIRED
  = QML 实例创建时的必填输入契约

QML default
  = 省略属性名的子对象接收入口

QML readonly
  = 不可重新赋值，但原绑定结果可以更新

C++ BINDABLE
  = 通过可绑定存储支持依赖追踪与绑定访问
```
