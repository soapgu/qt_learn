# Qt QML Signal 与分层事件流

## 1. 文档目标

本文以表决项目的 `ServiceControlPanel.qml` 为主线，解释 QML 自定义 signal、控件事件、ViewModel 命令、后端事件和属性通知在分层架构中的不同职责。

重点回答：

- `signal startRequested` 是什么；
- `onClicked`、`onStartRequested` 和 `startServer()` 如何串联；
- QML signal、`IVoteServer` signal 和 `Q_PROPERTY NOTIFY` 有何异同；
- 为什么可复用组件应该发出用户意图，而不是直接调用全局 ViewModel；
- Qt、WPF、Android View 和 Compose 如何表达同类事件流。

属性、`READ/WRITE/NOTIFY` 和双向绑定参见：[Qt Q_PROPERTY 与跨平台双向绑定对照](Qt%20Q_PROPERTY与跨平台双向绑定对照.md)。

官方参考：

- [Qt QML Signal and Handler Event System](https://doc.qt.io/qt-6/qtqml-syntax-signals.html)
- [Qt Connections QML Type](https://doc.qt.io/qt-6/qml-qtqml-connections.html)
- [Qt Signals & Slots](https://doc.qt.io/qt-6/signalsandslots.html)
- [WPF Commanding Overview](https://learn.microsoft.com/dotnet/desktop/wpf/advanced/commanding-overview)
- [Android Custom View Components](https://developer.android.com/develop/ui/views/layout/custom-views/custom-components)
- [Jetpack Compose State Hoisting](https://developer.android.com/develop/ui/compose/state-hoisting)

## 2. ServiceControlPanel 的输入和输出

当前组件：

```qml
RowLayout {
    id: root

    property bool running: false

    signal startRequested
    signal stopRequested

    Button {
        text: qsTr("启动服务")
        enabled: !root.running
        onClicked: root.startRequested()
    }

    Button {
        text: qsTr("停止服务")
        enabled: root.running
        onClicked: root.stopRequested()
    }
}
```

从组件 API 的角度理解：

```text
ServiceControlPanel
  输入：running
  输出：startRequested、stopRequested
```

`running` 是渲染所需的当前状态，两个 signal 是组件向外报告的用户意图：

```text
状态向下
VoteServerViewModel.running
    ↓
ServiceControlPanel.running
    ↓
Button.enabled

事件向上
用户点击 Button
    ↓
ServiceControlPanel.startRequested
    ↓
VoteServerViewModel.startServer()
```

## 3. QML signal 的声明、发送和监听

### 3.1 无参数 signal

```qml
signal startRequested
signal stopRequested
```

也可以写成：

```qml
signal startRequested()
```

没有参数时两种形式含义相同。QML 没有单独的 `emit` 关键字，调用信号本身就表示发送：

```qml
root.startRequested()
```

近似于 C++：

```cpp
emit startRequested();
```

### 3.2 带参数 signal

```qml
signal portChangedByUser(int port)
signal submitRequested(string userName, int port)
```

发送：

```qml
root.portChangedByUser(30000)
root.submitRequested("张三", 30000)
```

监听：

```qml
SomeComponent {
    onPortChangedByUser: port => viewModel.updatePort(port)

    onSubmitRequested: (userName, port) => {
        viewModel.submit(userName, port)
    }
}
```

参数只描述这一次事件携带的数据，不会让 signal 变成状态存储。

### 3.3 on<SignalName> 处理器

声明一个 signal 后，QML 自动提供对应的处理器名称：

| signal | 处理器 |
| --- | --- |
| `startRequested` | `onStartRequested` |
| `stopRequested` | `onStopRequested` |
| `submitRequested` | `onSubmitRequested` |

主页面因此可以写：

```qml
ServiceControlPanel {
    running: voteViewModel.running
    onStartRequested: voteViewModel.startServer()
    onStopRequested: voteViewModel.stopServer()
}
```

作用近似于：

```cpp
connect(controlPanel,
        &ServiceControlPanel::startRequested,
        viewModel,
        &VoteServerViewModel::startServer);
```

## 4. 从 Button.clicked 到 ViewModel 命令

### 4.1 提升内部控件事件

```qml
Button {
    onClicked: root.startRequested()
}
```

执行过程：

```text
Button 发出 clicked
    ↓
执行 onClicked
    ↓
发出根组件的 startRequested
```

它把具体控件事件提升为组件级用户意图：

```text
Button.clicked
  = 某个按钮被点击

ServiceControlPanel.startRequested
  = 用户请求启动服务
```

外部不必知道内部使用的是 `Button`、`ToolButton`、`TapHandler`、快捷键还是菜单项。

### 4.2 为什么叫 Requested

组件无法保证启动成功，只能表达请求：

| 名称 | 含义 |
| --- | --- |
| `startRequested` | 用户希望业务层尝试启动 |
| `startServer()` | ViewModel 执行启动命令 |
| `runningChanged()` | 当前运行状态已经变化 |
| `controllerConnected()` | 后端发生控制器连接事件 |

启动可能成功、失败、被拒绝或进入异步流程，所以 View 不能提前发送 `serverStarted`。

### 4.3 Main.qml 负责组装

```qml
ServiceControlPanel {
    running: voteViewModel.running
    onStartRequested: voteViewModel.startServer()
    onStopRequested: voteViewModel.stopServer()
}
```

组件不知道 ViewModel 类型、后端类型、是否需要确认或失败时如何处理。它只负责显示控件并报告用户意图。

## 5. 为什么不直接调用全局 ViewModel

不推荐在可复用组件内部写：

```qml
Button {
    onClicked: voteViewModel.startServer()
}
```

这会让组件隐式依赖同名 context property，导致：

- 无法脱离应用组合根独立创建；
- 测试必须构造同名全局对象；
- ViewModel 改名或更换注入方式时需要修改组件；
- 组件混入业务调用职责；
- 难以插入确认、权限检查或导航；
- 工具难以静态推断 context property 类型。

通过 signal，调用方可选择不同处理策略：

```qml
ServiceControlPanel {
    onStartRequested: voteViewModel.startServer()
}
```

也可以改成：

```qml
ServiceControlPanel {
    onStartRequested: confirmationDialog.open()
}
```

组件内部将来即使替换控件，对外 API 仍然不变。

## 6. 三类 signal 的分层职责

### 6.1 View 层用户意图

```qml
signal startRequested
signal stopRequested
```

```text
方向：View → ViewModel
语义：用户想做什么
```

### 6.2 后端事实事件

```cpp
signals:
    void controllerConnected(const QString &ip);
    void controllerDisconnected(const QString &ip);
    void errorOccurred(const QString &context,
                       const QString &message);
```

```text
方向：Backend → ViewModel
语义：SDK 或设备刚刚发生了什么
```

### 6.3 ViewModel 属性通知

```cpp
Q_PROPERTY(bool running READ running NOTIFY runningChanged)

signals:
    void runningChanged();
```

```text
方向：ViewModel → View
语义：可持续读取的当前状态已经变化
```

统一对照：

| signal | 所在层 | 语义 | 是否保存状态 |
| --- | --- | --- | --- |
| `Button.clicked` | 控件内部 | 原始交互事件 | 否 |
| `startRequested` | QML 组件 | 用户意图 | 否 |
| `controllerConnected` | 后端 | 已发生的设备事实 | 否 |
| `runningChanged` | ViewModel | `running` 属性变化通知 | 状态由属性保存 |
| `serviceStateChanged` | ViewModel | 状态属性变化通知 | 状态由属性保存 |

前三者是瞬时事件。后两者底层也是 signal，但通过 `Q_PROPERTY NOTIFY` 与可重新读取的状态关联。

## 7. 完整事件循环

### 7.1 用户启动服务

```text
用户点击“启动服务”
    ↓
Button.clicked
    ↓
ServiceControlPanel.startRequested
    ↓
Main.qml.onStartRequested
    ↓
VoteServerViewModel.startServer()
    ↓
IVoteServer.start("", 30000)
    ↓
VoteServerViewModel.setRunning(true)
    ↓
emit runningChanged()
    ↓
QML 重新读取 voteViewModel.running
    ↓
按钮状态刷新
```

### 7.2 控制器连接

```text
真实控制器或 Mock 发生连接
    ↓
IVoteServer.controllerConnected(ip)
    ↓
VoteServerViewModel 接收事件
    ├── setServiceState(Connected, ip)
    └── appendLog("控制器已连接")
    ↓
emit serviceStateChanged()
emit logEntriesChanged()
    ↓
QML 状态面板和日志列表刷新
```

### 7.3 分层方向图

```text
用户
  ↓ 点击
QML 内部控件事件
  ↓ 提升为用户意图
QML 组件 signal
  ↓
ViewModel 命令
  ↓ 调用
IVoteServer
  ↓ 后端事实事件
ViewModel 状态转换
  ↓ 属性通知
QML 属性绑定
  ↓
用户看到新状态
```

## 8. signal 不保存状态或历史

执行 `root.startRequested()` 只会通知当时已经连接的处理者。后来创建的对象不会自动收到过去的请求。

| 需求 | 推荐建模 |
| --- | --- |
| 当前是否运行 | `property`/`Q_PROPERTY` |
| 用户刚刚请求启动 | signal |
| 启停请求历史 | Model 或集合 |
| 最近一次错误 | 属性 |
| 每一次错误记录 | 日志 Model |

判断原则：

> 新订阅者加入后，是否必须立即知道当前值？

- 如果是，使用属性或 Model；
- 如果否，只关心事件发生，使用 signal。

## 9. Connections 的使用场景

父组件直接处理直属子组件事件时，优先使用内联处理器：

```qml
ServiceControlPanel {
    onStartRequested: voteViewModel.startServer()
}
```

目标对象无法方便地写在声明位置、需要多个连接或目标动态变化时，可以使用：

```qml
ServiceControlPanel {
    id: controlPanel
}

Connections {
    target: controlPanel

    function onStartRequested() {
        voteViewModel.startServer()
    }

    function onStopRequested() {
        voteViewModel.stopServer()
    }
}
```

两种形式使用同一信号机制。不要为了形式统一，把简单直属事件全部改成 `Connections`。

## 10. 事件命名与重复防护

推荐使用能表达意图或事实的名称：

| 后缀 | 适用语义 |
| --- | --- |
| `Requested` | 用户请求执行操作 |
| `Selected` | 用户完成选择 |
| `Clicked` | 控件被点击 |
| `Changed` | 属性或值已经变化 |
| `Occurred` | 错误等事件已经发生 |

View 层通过状态禁用重复操作：

```qml
Button {
    enabled: !root.running
}
```

ViewModel 仍需独立保护：

```cpp
void VoteServerViewModel::startServer()
{
    if (m_running) {
        return;
    }

    // 调用后端启动
}
```

View 避免用户触发无效操作，ViewModel 则保证来自按钮、快捷键、测试或其他页面的调用都安全。

## 11. WPF 对照

### 11.1 UserControl 自定义事件

```csharp
public partial class ServiceControl : UserControl
{
    public event EventHandler? StartRequested;

    private void StartButton_Click(object sender, RoutedEventArgs e)
    {
        StartRequested?.Invoke(this, EventArgs.Empty);
    }
}
```

对应 QML：

```qml
signal startRequested

Button {
    onClicked: root.startRequested()
}
```

### 11.2 ICommand

完整 WPF MVVM 更常使用：

```xml
<Button Content="启动服务"
        Command="{Binding StartServerCommand}" />
```

当前项目用组件 signal 接到 ViewModel 方法：

```qml
ServiceControlPanel {
    onStartRequested: voteViewModel.startServer()
}
```

```text
QML 组件 signal
  ≈ UserControl 自定义事件

主页面把 signal 接到 ViewModel
  ≈ View 把事件或 ICommand 接到 ViewModel
```

## 12. Android View 与 Compose 对照

### 12.1 Android 自定义 View 回调

```kotlin
class ServiceControlView : LinearLayout {
    var onStartRequested: (() -> Unit)? = null

    private fun handleStartClick() {
        onStartRequested?.invoke()
    }
}
```

```kotlin
controlView.onStartRequested = {
    viewModel.startServer()
}
```

对应：

```qml
ServiceControlPanel {
    onStartRequested: voteViewModel.startServer()
}
```

### 12.2 Compose 事件 Lambda

```kotlin
@Composable
fun ServiceControlPanel(
    running: Boolean,
    onStartRequested: () -> Unit,
    onStopRequested: () -> Unit
) {
    Button(
        enabled = !running,
        onClick = onStartRequested
    ) {
        Text("启动服务")
    }
}
```

调用：

```kotlin
ServiceControlPanel(
    running = state.running,
    onStartRequested = viewModel::startServer,
    onStopRequested = viewModel::stopServer
)
```

| QML | Compose |
| --- | --- |
| `property bool running` | `running: Boolean` |
| `signal startRequested` | `onStartRequested: () -> Unit` |
| `enabled: !root.running` | `enabled = !running` |
| `onClicked: root.startRequested()` | `onClick = onStartRequested` |
| `onStartRequested: vm.startServer()` | `onStartRequested = vm::startServer` |

两者都体现“状态向下、事件向上”。

## 13. 统一对照

| 环节 | Qt/QML | WPF | Android View | Compose |
| --- | --- | --- | --- | --- |
| 状态输入 | QML property | DependencyProperty/绑定属性 | View 属性 | 函数参数 |
| 内部交互 | `Button.clicked` | `Button.Click` | `OnClickListener` | `Button.onClick` |
| 组件输出 | QML signal | event/RoutedEvent | callback/listener | 事件 Lambda |
| 业务入口 | `Q_INVOKABLE` 方法 | `ICommand`/方法 | ViewModel 方法 | ViewModel 方法 |
| 后端事件 | C++ signal | C# event | callback/Flow | Flow |
| 状态通知 | `Q_PROPERTY NOTIFY` | `INotifyPropertyChanged` | LiveData/StateFlow | State/StateFlow |

这是职责对照，不代表生命周期、线程、重放、冒泡或背压语义完全一致。

## 14. 项目实践准则

1. 可复用 QML 组件通过 property 接收状态，通过 signal 输出用户意图。
2. 内部控件事件先提升为语义化组件事件，不直接依赖全局 ViewModel。
3. 意图信号使用 `Requested`，不要提前命名为已经完成的业务结果。
4. 直属子组件事件使用内联 `on<Signal>`；复杂或动态目标使用 `Connections`。
5. signal 不保存状态；当前值用属性，历史记录用 Model。
6. ViewModel 独立防止重复命令，不能只依赖按钮禁用。
7. `IVoteServer` 报告后端事实，ViewModel 转换状态，QML 不维护 SDK 状态机。
8. 带参数 signal 只携带该次事件需要的最小数据，不暴露后端对象。

快速定位职责：

```text
Button.clicked
  = 内部控件事件

ServiceControlPanel.startRequested
  = View 层用户意图

VoteServerViewModel.startServer()
  = 处理意图的命令

IVoteServer.controllerConnected(ip)
  = 后端已经发生的事实

VoteServerViewModel.runningChanged()
  = 当前属性变化通知
```

## 15. 总结

`ServiceControlPanel.qml` 的 signal 是组件公开 API 的一部分：

```text
输入状态：running
输出事件：startRequested、stopRequested
```

它把具体按钮点击提升为稳定的用户意图，使组件不依赖 ViewModel，同时由主页面负责组装业务处理者。

```text
QML signal
  View → ViewModel：用户想做什么

IVoteServer signal
  Backend → ViewModel：后端发生了什么

Q_PROPERTY NOTIFY signal
  ViewModel → View：当前状态变了
```

三者使用同一套 Qt 信号机制，却分别构成清晰的输入、处理和反馈闭环。
