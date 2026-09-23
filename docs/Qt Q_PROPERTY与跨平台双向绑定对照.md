# Qt Q_PROPERTY 与跨平台双向绑定对照

## 1. 文档目标

本文面向熟悉 WPF 或 Android MVVM、正在学习 Qt Quick/QML 的开发者，系统说明：

- `Q_PROPERTY` 的 `READ`、`WRITE`、`NOTIFY`、`CONSTANT` 如何协作；
- `NOTIFY` 后面的名称如何与 `signals` 中的信号关联；
- C++ 状态如何驱动 QML Binding 自动刷新；
- QML 如何把用户输入写回 C++；
- Qt、WPF、Android XML Data Binding 和 Jetpack Compose 如何表达同类数据流。

本文以当前项目的 `VoteServerViewModel` 为主线。不同框架之间只是职责近似，不代表类型、线程、生命周期或运行机制完全相同。

QML 组件自定义 signal、用户意图和分层事件流参见：[Qt QML Signal 与分层事件流](Qt%20QML%20Signal与分层事件流.md)。

`moc`、`Q_OBJECT`、`Q_GADGET`、`Q_ENUM` 和 QML 类型注册的基础原理参见：[Qt 元对象系统与 QML 类型注册](Qt元对象系统与QML类型注册.md)。

官方参考：

- [Qt Property System](https://doc.qt.io/qt-6/properties.html)
- [Qt：向 QML 暴露 C++ 属性](https://doc.qt.io/qt-6/qtqml-cppintegration-exposecppattributes.html)
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

它不会自动生成：

- `m_running` 成员变量；
- `running()` Getter；
- `runningChanged()` 信号；
- 修改属性或发送通知的代码。

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

## 4. READ、WRITE、NOTIFY 和 CONSTANT

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

`NOTIFY` 不负责携带和保存状态。它的主要职责是告诉依赖方：

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
  = 构造完成后不再变化

NOTIFY / WRITE
  = 属性可能变化
```

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

## 8. WPF 对照

### 8.1 INotifyPropertyChanged

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

### 8.2 OneWay 与 TwoWay

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

### 8.3 UpdateSourceTrigger 与 QML 写回时机

| WPF | QML 近似行为 |
| --- | --- |
| `PropertyChanged` | `onTextEdited` 每次编辑写回 |
| `LostFocus` | `onEditingFinished` 写回 |
| `Explicit` | 保存按钮调用命令写回 |

### 8.4 DependencyProperty

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

## 9. Android XML Data Binding 对照

### 9.1 @Bindable 与通知

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

### 9.2 @={} 双向绑定

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

### 9.3 MutableLiveData

```kotlin
class UserViewModel : ViewModel() {
    val userName = MutableLiveData("")
}
```

```xml
<EditText android:text="@={viewModel.userName}" />
```

`MutableLiveData` 同时具有当前值、可写入口和观察通知，因此在职责上接近包含 `READ + WRITE + NOTIFY` 的 Qt 属性。但 `LiveData` 具有 Android 生命周期感知语义，普通 `QObject` 和 Qt 信号本身没有完全相同的能力。

## 10. Jetpack Compose 对照

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

## 11. 四套实现的统一对照

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

## 12. 常见错误

### 12.1 修改成员变量但漏发 NOTIFY

错误：

```cpp
m_running = true;
```

结果是 C++ 中的值已经变化，QML 仍显示旧状态。应统一通过 Setter：

```cpp
setRunning(true);
```

### 12.2 值没有变化也发送信号

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

### 12.3 把 WRITE 当成自动 TwoWay Binding

```cpp
Q_PROPERTY(QString name READ name WRITE setName NOTIFY nameChanged)
```

只是让 QML 可以写入 `name`。要形成双向交互，还需要选择输入事件并显式写回。

### 12.4 使用 onTextChanged 无差别写回

程序刷新文本也可能触发 `textChanged`。如果目标是响应用户编辑，优先使用：

```qml
onTextEdited: viewModel.name = text
```

### 12.5 直接写业务状态

不要为了方便双向绑定，把 `running`、`connected` 或 `serviceState` 暴露为可写属性。它们是业务操作结果，不是用户表单输入。

### 12.6 所有属性共用一个通知信号

Qt 允许共享 `NOTIFY`，但大范围共享会使无关绑定一起重新计算。只有属性总是一起变化、使用频率较低且语义明确时才共享。

## 13. 项目实践准则

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
```
