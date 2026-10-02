# Qt Q_INVOKABLE 与跨平台命令调用对照

## 1. 文档目标

本文面向熟悉 WPF、Caliburn.Micro 或 Android MVVM、正在学习 Qt Quick/QML 的开发者，以 Qt 6.8.3 和表决系统的启动、停止服务为背景，梳理：

- `Q_INVOKABLE` 如何让 C++ 方法进入 Qt 元对象系统；
- 方法暴露、QML 类型注册和对象实例注入分别解决什么问题；
- 普通方法、slot、signal、属性和界面命令的边界；
- 参数、返回值、动态调用、线程与异步反馈；
- WPF `ICommand`、Caliburn.Micro Action、Android XML Data Binding 与 Compose 的对应写法。

本文中的代码均为教学示例。`VoteServerViewModel` 名称沿用已有学习资料，但下面的实现只模拟状态切换，不监听真实端口，也不连接供应商 SDK。`configurePort()`、`describePort()`、命令可用性属性及后续异步接口是教学扩展，不表示现有表决项目已经实现这些接口。

相关专题：

- [Qt 元对象系统与 QML 类型注册](Qt元对象系统与QML类型注册.md)：`moc`、`Q_OBJECT`、枚举和类型注册。
- [Qt Q_PROPERTY 与跨平台双向绑定对照](Qt%20Q_PROPERTY与跨平台双向绑定对照.md)：状态、属性读写和通知。
- [Qt QML Signal 与分层事件流](Qt%20QML%20Signal与分层事件流.md)：组件用户意图、后端事实与事件分层。

## 2. Q_INVOKABLE 解决什么问题

普通 C++ 调用在编译时就知道类型和方法：

```cpp
viewModel.startServer();
```

QML 需要通过 Qt 暴露的类型信息访问 C++ 对象。单纯把一个成员函数放进 `public:`，不会自动让它成为 QML 可调用的方法。

```cpp
class Example : public QObject
{
    Q_OBJECT

public:
    void internalOperation();             // 普通 C++ 公有方法
    Q_INVOKABLE void startServer();        // 登记到元对象的方法

public slots:
    void stopServer();                    // slot 也登记到元对象
};
```

`Q_INVOKABLE` 放在方法声明的返回类型前。`moc` 在构建时识别它，生成方法名称、参数、返回类型等元数据及调用分发代码；这些生成的 C++ 代码再参与编译和链接。

```text
头文件中的 Q_OBJECT、Q_INVOKABLE
    → moc 生成元对象信息和调用分发代码
    → 编译、链接
    → QML 或按名称的动态调用找到方法
    → 执行原来的 C++ 方法体
```

方法体仍需自己实现。这个宏不会生成业务逻辑，也不会自动建立按钮绑定、检查能否执行或启动后台线程。方法也不限于“命令”，可以是查询方法，只要暴露给 QML 的接口确实需要它。

Qt 官方列出的 QObject 方法暴露入口是公有 `Q_INVOKABLE` 方法和公有 slot；本文以 `QObject + Q_OBJECT` ViewModel 为主线。[Qt：向 QML 暴露方法](https://doc.qt.io/qt-6.8/qtqml-cppintegration-exposecppattributes.html#exposing-methods-including-qt-slots)

`Q_GADGET` 也能带有 `Q_INVOKABLE` 元数据，可通过适合 gadget 的 API 调用；它没有 QObject 的信号、线程亲和性或对象生命周期，不能直接套用本文的 ViewModel 方案。[Qt：Q_GADGET](https://doc.qt.io/qt-6.8/qobject.html#Q_GADGET)

## 3. 方法暴露、类型注册与实例注入

这三个步骤不是同一件事：

| 步骤 | 示例 | 解决的问题 |
| --- | --- | --- |
| 元对象方法登记 | `Q_OBJECT`、`Q_INVOKABLE`、`public slots` | 这个类有哪些可动态调用的方法 |
| QML 类型注册 | `QML_ELEMENT` 配合 `qt_add_qml_module()` | QML 模块中的类型名对应哪个 C++ 类型 |
| 实例创建与注入 | C++ 创建对象，`setInitialProperties()` 传入 | 当前页面具体操作哪一个实例 |

`QML_UNCREATABLE` 表示 QML 能认识类型，但不能自行创建实例；不妨碍调用已经注入的实例。`Q_INVOKABLE` 也不等于把类注册成 QML 类型。

本文采用强类型根属性：

```qml
required property VoteServerViewModel voteViewModel
```

C++ 在加载根组件前用 `setInitialProperties()` 设置这个属性。该 API 设置根对象的初始属性，不会把变量自动注入每一个子组件。[Qt：setInitialProperties](https://doc.qt.io/qt-6.8/qqmlapplicationengine.html#setInitialProperties)

已有资料也出现过 `setContextProperty("voteViewModel", ...)`。它能暴露实例，但与强类型根属性的作用范围、工具可见性不同；不要把两种注入方式在一个例子里混用。注册类型和注入实例的详细边界见元对象专题。

### 3.1 最小示例：C++ ViewModel

下面四个文件组成一个独立的教学工程；本仓库仍只保存文档，没有新增应用工程。

`VoteServerViewModel.h`：

```cpp
#pragma once

#include <QObject>
#include <QString>
#include <QtQml/qqmlregistration.h>

class VoteServerViewModel : public QObject
{
    Q_OBJECT
    QML_ELEMENT
    QML_UNCREATABLE("由应用程序创建")
    Q_PROPERTY(bool running READ running NOTIFY runningChanged)
    Q_PROPERTY(bool canStart READ canStart NOTIFY runningChanged)
    Q_PROPERTY(bool canStop READ canStop NOTIFY runningChanged)
    Q_PROPERTY(int port READ port NOTIFY portChanged)

public:
    explicit VoteServerViewModel(QObject *parent = nullptr)
        : QObject(parent) {}

    bool running() const { return m_running; }
    bool canStart() const { return !m_running; }
    bool canStop() const { return m_running; }
    int port() const { return m_port; }

    Q_INVOKABLE void startServer()
    {
        if (!canStart()) return;
        // 教学模拟：真实项目应调用 Service，并依据后端结果更新状态。
        setRunning(true);
    }

    Q_INVOKABLE void stopServer()
    {
        if (!canStop()) return;
        setRunning(false);
    }

    Q_INVOKABLE bool configurePort(int port)
    {
        if (m_running || port < 1 || port > 65535) return false;
        if (m_port == port) return true;
        m_port = port;
        emit portChanged();
        return true;
    }

    Q_INVOKABLE QString describePort(const QString &prefix) const
    {
        return prefix + QString::number(m_port);
    }

signals:
    void runningChanged();
    void portChanged();

private:
    void setRunning(bool running)
    {
        if (m_running == running) return;
        m_running = running;
        emit runningChanged();
    }

    bool m_running = false;
    int m_port = 9000;
};
```

`running`、`canStart`、`canStop` 共享 `runningChanged`：三个 Getter 都依赖 `m_running`。它们是只读属性，操作通过方法完成；宏不会替你推导或发送变化通知。

本例没有把 C++ 类标记为 `final`。Qt 6.8.3 的这条 QML 注册路径会实例化继承该类型的包装模板，即使使用 `QML_UNCREATABLE`，`final` 也会导致编译失败。不可由 QML 创建与禁止 C++ 继承是两种不同限制。

### 3.2 最小示例：创建并注入实例

`main.cpp`：

```cpp
#include <QGuiApplication>
#include <QQmlApplicationEngine>
#include <QVariant>
#include "VoteServerViewModel.h"

int main(int argc, char *argv[])
{
    QGuiApplication app(argc, argv);

    VoteServerViewModel viewModel;
    QQmlApplicationEngine engine;
    engine.setInitialProperties({
        {QStringLiteral("voteViewModel"), QVariant::fromValue(&viewModel)}
    });
    engine.loadFromModule("VoteDevice", "Main");
    if (engine.rootObjects().isEmpty()) return 1;

    return app.exec();
}
```

声明顺序使 engine 先销毁、ViewModel 后销毁，保证 QML 使用对象期间实例仍然有效。这个栈对象由 C++ 管理；不要返回已经销毁的局部对象地址。

### 3.3 最小示例：QML 调用

`Main.qml`：

```qml
import QtQuick
import QtQuick.Controls
import QtQuick.Layouts
import VoteDevice

ApplicationWindow {
    id: root
    required property VoteServerViewModel voteViewModel

    visible: true
    width: 360
    height: 180
    title: qsTr("服务调用示例")

    ColumnLayout {
        anchors.centerIn: parent

        Label {
            text: root.voteViewModel.running ? qsTr("运行中") : qsTr("已停止")
        }

        RowLayout {
            Button {
                text: qsTr("启动服务")
                enabled: root.voteViewModel.canStart
                onClicked: root.voteViewModel.startServer()
            }
            Button {
                text: qsTr("停止服务")
                enabled: root.voteViewModel.canStop
                onClicked: root.voteViewModel.stopServer()
            }
        }
    }
}
```

这里有两条独立的数据流：

```text
点击 → onClicked → Q_INVOKABLE 方法 → 修改状态
状态变化 → runningChanged → 属性绑定重新求值 → 更新文字和 enabled
```

可复用组件继续使用现有事件专题中的方式：组件发出 `startRequested`，主页面用 `onStartRequested: root.voteViewModel.startServer()` 连接业务处理者。调用入口相同，只是 View 层多了一层语义事件。

### 3.4 最小示例：构建配置

`CMakeLists.txt`：

```cmake
cmake_minimum_required(VERSION 3.21)
project(InvokableDemo LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

find_package(Qt6 6.8 REQUIRED COMPONENTS Core Gui Qml Quick QuickControls2)
qt_standard_project_setup(REQUIRES 6.8)

qt_add_executable(invokable_demo main.cpp)
qt_add_qml_module(invokable_demo
    URI VoteDevice
    VERSION 1.0
    SOURCES VoteServerViewModel.h
    QML_FILES Main.qml
)
target_link_libraries(invokable_demo PRIVATE
    Qt6::Core Qt6::Gui Qt6::Qml Qt6::Quick Qt6::QuickControls2
)
```

`qt_standard_project_setup()` 启用 AUTOMOC；`qt_add_qml_module()` 将头文件纳入类型注册处理，并打包 QML 资源。本例直接将模块放在可执行目标中，不涉及已有项目的静态 UI 库拆分。[Qt：项目设置](https://doc.qt.io/qt-6.8/qt-standard-project-setup.html)、[Qt：QML 模块构建](https://doc.qt.io/qt-6.8/qt-add-qml-module.html)

## 4. 普通方法、slot、signal、属性如何选择

| 声明 | 普通 C++ 调用 | 元对象中的身份 | QML 中的主要用途 |
| --- | --- | --- | --- |
| 普通 `public` 方法 | 可以 | 不因 public 自动登记为方法 | 不自动成为可调用方法 |
| 公有 `Q_INVOKABLE` 方法 | 可以 | `QMetaMethod::Method` | 主动调用操作或查询 |
| `public slots` 方法 | 可以 | `QMetaMethod::Slot` | 主动调用，也可接收信号 |
| `signals` 方法 | C++ 用 `emit` 发出 | `QMetaMethod::Signal` | 订阅通知；QML 也能触发可见信号，但不能代替业务命令 |
| `Q_PROPERTY` | 通过 Getter/Setter 访问 | `QMetaProperty` | 读取、写入和绑定当前状态 |

属性 Getter/Setter 不因出现在 `READ/WRITE` 中就自动变成独立的 QML 方法；`vm.running` 和 `vm.running()` 不是同一个接口。

项目可以约定“界面主动操作使用 `Q_INVOKABLE`，接收其他对象通知使用 slot”，便于阅读，但这是语义约定。公有 slot 同样能从 QML 调用，invokable 方法也能用现代函数指针 `connect()` 接收信号。

```cpp
// 普通 public 成员函数也可以作为接收端，不要求 Q_INVOKABLE 或 slots。
QObject::connect(service, &Service::stopped,
                 &viewModel, &ViewModel::handleStopped);
```

上面是结构示意，假设 `Service`、`ViewModel` 均已定义、参数兼容且方法可访问。函数指针形式让编译器检查方法；旧式 `SIGNAL/SLOT` 字符串连接依赖元对象中的 signal/slot 信息，不能机械替换。[Qt：信号与槽](https://doc.qt.io/qt-6.8/signalsandslots.html)

## 5. 参数、返回值和类型转换

### 5.1 无参调用与带参调用

第 3 节的 ViewModel 可以直接使用：

```qml
root.voteViewModel.startServer()
root.voteViewModel.stopServer()

const accepted = root.voteViewModel.configurePort(9100)
if (!accepted) {
    console.warn("当前状态或端口不允许修改")
}

const description = root.voteViewModel.describePort("端口：")
console.log(description)
```

这些语句应放在事件处理器或函数中。当前状态仍通过 `port`、`running` 属性读取；不要为了读取状态反复调用带副作用的方法。

`configurePort()` 返回 `true` 的含义是“端口配置被接受”，不是“服务已经监听”。布尔返回值的业务含义由接口定义，宏本身没有约定。

查询方法也不自动建立状态依赖。例如 `text: vm.describePort("端口：")` 中，QML 不会因为 C++ 方法体读取了 `m_port` 就订阅 `portChanged`。需要随端口变化刷新的文本，可写 `text: "端口：" + vm.port`，或公开带 NOTIFY 的计算属性。

### 5.2 常用类型

| C++ 接口类型 | QML/JavaScript 侧常见表现 | 注意事项 |
| --- | --- | --- |
| `bool` | 布尔值 | 适合简单同步判断 |
| `int`、`double` | 数值 | 整数范围、精度和业务范围需要检查 |
| `QString` | 字符串 | QML 会做支持的类型转换 |
| `QStringList` | 支持的字符串序列 | 不要把它当成任意 C++ 容器的自动导出规则 |
| `QVariantList`、`QVariantMap` | 数组／对象式数据 | 元素和键值也须是可转换的数据 |
| 已暴露的 `QObject *` | 对象引用 | 需要明确生命周期和所有权 |
| 已正确导出的枚举 | 具名枚举值 | 优先使用枚举常量，避免魔法数字 |

QML 的类型转换与 C++ 按名称调用的参数匹配不是同一套规则。也不能认为 `Q_DECLARE_METATYPE` 或 `qRegisterMetaType()` 会自动把任意 C++ 类型注册为可创建的 QML 类型。[Qt：QML 与 C++ 类型转换](https://doc.qt.io/qt-6.8/qtqml-cppintegration-data.html)

如果 invokable 方法返回无父对象的 `QObject *`，QML 的所有权规则可能使对象交给引擎管理。本文没有这种返回接口；真实项目需确认 parent、对象所有权和调用方使用期限，不能把所有 QObject 指针都视为 C++ 永久持有。

### 5.3 重载和默认参数

QML 支持暴露的重载，但 JavaScript 的数值和动态类型可能使选择结果不够直观。例如同时暴露 `setValue(int)` 和 `setValue(double)`，调用方容易误判实际入口。

面向界面的 API 优先使用语义明确的名称，例如 `configurePort(int)`、`configureAddress(QString)`，或单一明确签名。默认参数在部分元对象/QML 调用场景中能使用，但不能据此认为所有调用路径都会进行与普通 C++ 一样的默认参数补全；示例显式传参，按名称调用时核对实际元对象签名。

## 6. QMetaObject::invokeMethod：按名称调用

### 6.1 同步调用并获取返回值

下面使用 Qt 6.5 起提供的参数推导重载，适用于本文 Qt 6.8.3 基线：

```cpp
VoteServerViewModel viewModel;
bool accepted = false;

const bool invoked = QMetaObject::invokeMethod(
    &viewModel, "configurePort", Qt::DirectConnection,
    qReturnArg(accepted), int{9100});

// invoked：是否成功找到并调用匹配方法。
// accepted：configurePort() 返回的业务结果。
```

例如传入端口 `0`，可以出现 `invoked == true`、`accepted == false`：方法调用成功，但业务校验拒绝。

返回字符串：

```cpp
QString description;
const bool invoked = QMetaObject::invokeMethod(
    &viewModel, "describePort", Qt::DirectConnection,
    qReturnArg(description), QStringLiteral("端口："));
```

传入的是方法名，不是 `"configurePort(int)"`；参数数量、类型及返回类型须匹配。这里显式传 `QString`，不能期待字符串字面量 `const char *` 像 QML 一样自动匹配 `QString`。运行前需确认直接调用符合对象的线程使用约束。[Qt：invokeMethod](https://doc.qt.io/qt-6.8/qmetaobject.html#invokeMethod)

### 6.2 查询元对象签名

```cpp
const QMetaObject *meta = viewModel.metaObject();
const int index = meta->indexOfMethod("configurePort(int)");
if (index >= 0) {
    const QMetaMethod method = meta->method(index);
    qDebug() << method.methodSignature() << method.methodType();
}
```

该片段另需包含 `<QMetaMethod>` 和 `<QDebug>`。与 `invokeMethod()` 不同，`indexOfMethod()` 查找的是规范化的完整签名。调试时先确认方法是否进入元对象，再检查参数匹配。

### 6.3 接受可调用对象的重载

`invokeMethod()` 还有接受 lambda／可调用对象的重载：

```cpp
QMetaObject::invokeMethod(
    &viewModel,
    [&viewModel] { viewModel.startServer(); },
    Qt::QueuedConnection);
```

这里没有按字符串查找 `startServer`。lambda 中是普通 C++ 调用，因此即使目标成员是普通公有方法，也不要求 `Q_INVOKABLE`。传入的 context 决定队列投递的执行上下文；它必须有效，捕获的其他对象也须有足够长的生命周期。

```text
invokeMethod(obj, "methodName", ...)
    → 查元对象，要求方法已登记且签名匹配

invokeMethod(context, callable, ...)
    → 调用已提供的可调用对象，不靠方法名反射
```

这是 Qt 6.8 支持的两类 API；不要把“使用了 invokeMethod”理解为“目标成员一定要 Q_INVOKABLE”。

## 7. 线程、队列与异步边界

### 7.1 Q_INVOKABLE 不切换线程

QML 中普通的 `vm.startServer()` 调用是同步调用。QML/Quick ViewModel 通常留在引擎所在的 GUI 线程；方法如果等待 SDK、网络响应或文件处理，仍会阻塞界面。

即使给 QObject 调用了 `moveToThread()`，直接 C++ 成员调用也不会自动切到对象所属线程。不要让 QML 直接持有 Worker，然后期待 `Q_INVOKABLE` 自动完成跨线程派发。

| 调用方式 | 执行位置／时机 | 调用方是否等待 |
| --- | --- | --- |
| 普通 C++ 调用／QML 普通方法调用 | 在调用线程直接执行 | 等待方法返回 |
| `Qt::DirectConnection` | 在调用线程立即执行 | 等待方法返回 |
| `Qt::QueuedConnection` | 投递到接收对象所属线程的事件循环 | 通常立即返回，之后执行 |
| `Qt::AutoConnection` | 当前调用线程与接收对象线程相同则直接，否则队列 | 取决于实际选择 |
| `Qt::BlockingQueuedConnection` | 投递到接收对象线程并等待完成 | 阻塞；同线程使用会死锁 |

投递到 GUI 线程的重任务仍然会卡住 GUI，只是延后执行。队列调用通常要求接收线程有运行中的事件循环；目标销毁或线程退出也可能使工作不再执行。队列投递成功不等于业务完成。

### 7.2 按名称排队调用

```cpp
const bool queued = QMetaObject::invokeMethod(
    &viewModel, "stopServer", Qt::QueuedConnection);
```

队列调用不使用 `qReturnArg()` 同步取得业务返回值；需要结果时，通过后续状态或完成信号报告。按名称异步传参需要可复制、完整定义的类型；Qt 6.5 起的模板重载会自动注册所用类型，但不支持只前置声明的参数类型或非 const 引用参数。

### 7.3 推荐的异步职责划分

下面是架构示意，未提供完整 Worker、线程初始化和关闭代码：

```text
QML 点击启动
    → GUI 线程中的 ViewModel.startServer()
    → 校验状态，设置 busy=true，发送 NOTIFY
    → Service/Worker 异步处理真实启动
    → 将成功／失败结果投递回 GUI 线程
    → ViewModel 更新 running、busy、errorText
    → Q_PROPERTY 通知刷新界面，必要时发完成信号
```

ViewModel 可以公开：

```cpp
// 独立的异步接口示意，不是第 3 节同步模拟类的追加实现。
Q_PROPERTY(bool busy READ busy NOTIFY busyChanged)
Q_PROPERTY(bool canStart READ canStart NOTIFY canStartChanged)
Q_INVOKABLE void startServer();

signals:
    void busyChanged();
    void canStartChanged();
    void startFinished(bool success, const QString &message);
```

异步场景的 `canStart` 应根据服务状态和 `busy` 计算，不能只判断 `!running`；从 Starting 到 Listening 之间也必须防止重复启动。每个依赖项变化都要通知命令可用性属性。

QML 可用 `Connections` 处理一次完成事件：

```qml
Connections {
    target: root.voteViewModel
    function onStartFinished(success, message) {
        if (!success) console.warn(message)
    }
}
```

此片段仅适用于上面的异步接口，不适用于第 3 节的同步模拟类。`startFinished` 表达这一次操作的结果，不保存当前状态；`running/busy/errorText` 则供界面随时读取。复杂状态机继续放在 Service 层，QML 不负责维护供应商 SDK 状态。

Qt 的线程亲和性与连接类型见：[Threads and QObjects](https://doc.qt.io/qt-6.8/threads-qobject.html)。

## 8. WPF：ICommand 不只是一个可调用方法

WPF MVVM 常将按钮绑定到命令对象：

```xml
<StackPanel>
    <Button Content="启动服务" Command="{Binding StartServerCommand}" />
    <Button Content="停止服务" Command="{Binding StopServerCommand}" />
</StackPanel>
```

假设 `DataContext` 已设置为 ViewModel。`ICommand` 有三个成员：

| WPF 成员 | 职责 | Qt/QML 的近似组合 |
| --- | --- | --- |
| `Execute(parameter)` | 执行操作 | `Q_INVOKABLE startServer()` |
| `CanExecute(parameter)` | 判断当前能否执行 | `canStart` 属性／状态判断 |
| `CanExecuteChanged` | 提示调用方重新检查可用性 | `canStart` 的 `NOTIFY` |

WPF 的 Button 命令源会查询命令是否可执行，并相应更新可用状态；Qt 的 Button 不会从一个 invokable 方法自动推导这些信息。因此在第 3 节中需要显式写 `enabled: vm.canStart`。

### 8.1 教学用命令对象

下面提供一个无参数版本，避免依赖未定义的第三方 `RelayCommand`。示例使用支持可空引用类型的 C# 语法；需要带参时可扩展委托签名，并绑定 `CommandParameter`。

```csharp
using System;
using System.Windows.Input;

public sealed class RelayCommand : ICommand
{
    private readonly Action execute;
    private readonly Func<bool> canExecute;

    public RelayCommand(Action execute, Func<bool> canExecute)
    {
        this.execute = execute;
        this.canExecute = canExecute;
    }

    public bool CanExecute(object? parameter) => canExecute();

    public void Execute(object? parameter)
    {
        if (CanExecute(parameter)) execute();
    }

    public event EventHandler? CanExecuteChanged;
    public void RaiseCanExecuteChanged() =>
        CanExecuteChanged?.Invoke(this, EventArgs.Empty);
}
```

`ICommand` 接口本身不强制 `Execute()` 再检查 `CanExecute()`；这里主动加上保护。它也不负责异步任务调度。

### 8.2 同一启动、停止场景

```csharp
using System.ComponentModel;

public sealed class VoteServerViewModel : INotifyPropertyChanged
{
    private bool running;
    public bool Running => running;

    public RelayCommand StartServerCommand { get; }
    public RelayCommand StopServerCommand { get; }
    public event PropertyChangedEventHandler? PropertyChanged;

    public VoteServerViewModel()
    {
        StartServerCommand = new RelayCommand(StartServer, () => !Running);
        StopServerCommand = new RelayCommand(StopServer, () => Running);
    }

    public void StartServer()
    {
        if (Running) return;
        SetRunning(true); // 教学模拟
    }

    public void StopServer()
    {
        if (!Running) return;
        SetRunning(false);
    }

    private void SetRunning(bool value)
    {
        if (running == value) return;
        running = value;
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(nameof(Running)));
        StartServerCommand.RaiseCanExecuteChanged();
        StopServerCommand.RaiseCanExecuteChanged();
    }
}
```

这个自定义命令显式发出 `CanExecuteChanged`，未使用 `CommandManager.RequerySuggested`。不要认为修改 ViewModel 属性就能自动触发所有自定义命令重新检查。

WPF 的 `RoutedCommand` 还有路由和 `CommandBinding` 机制；上面的 ViewModel 命令是自定义 `ICommand` 实现。`Q_INVOKABLE` 不包含 WPF 的命令路由能力。[WPF 命令概览](https://learn.microsoft.com/dotnet/desktop/wpf/advanced/commanding-overview)

## 9. Caliburn.Micro：Action 与守卫

如果熟悉 Caliburn.Micro，把 QML 调用理解为“View 触发 ViewModel 上的动作”会更直观：

```xml
<!-- 父视图已声明 xmlns:cal="http://www.caliburnproject.org"，
     且通过 Caliburn.Micro 绑定了 Action 目标。 -->
<Button Content="启动服务"
        cal:Message.Attach="[Event Click] = [Action StartServer]" />
<Button Content="停止服务"
        cal:Message.Attach="[Event Click] = [Action StopServer]" />
```

独立的 ViewModel 示例：

```csharp
using Caliburn.Micro;

public sealed class VoteServerViewModel : Screen
{
    private bool running;
    public bool Running => running;
    public bool CanStartServer => !running;
    public bool CanStopServer => running;

    public void StartServer()
    {
        if (!CanStartServer) return;
        SetRunning(true); // 教学模拟
    }

    public void StopServer()
    {
        if (!CanStopServer) return;
        SetRunning(false);
    }

    private void SetRunning(bool value)
    {
        if (running == value) return;
        running = value;
        NotifyOfPropertyChange(() => Running);
        NotifyOfPropertyChange(() => CanStartServer);
        NotifyOfPropertyChange(() => CanStopServer);
    }
}
```

Action 机制查找 `StartServer`，并识别对应的 `CanStartServer` 守卫。守卫可以是属性，也可以是与动作参数对应的布尔方法；属性守卫变化需要通知框架重新评估。[Caliburn.Micro Actions](https://caliburnmicro.com/documentation/actions)

```text
Caliburn.Micro：StartServer + CanStartServer + 属性通知 + Action 绑定
Qt/QML：startServer + canStart + NOTIFY + onClicked/组件事件连接
```

Qt 不会按 `Can<方法名>` 约定寻找守卫。只是给 C++ 加一个 `canStartServer()` 普通方法，不会自动禁用按钮。

Caliburn.Micro 还支持按控件名称约定动作。例如框架应用了 View 约定后，`x:Name="StartServer"` 的 Button 可以关联动作。它与显式 `Message.Attach` 是不同入口；同一按钮示例选择一种，避免重复配置。Qt 示例使用显式事件连接，不引入这种名称约定。

## 10. Android：Java/XML Data Binding

Android XML Data Binding 的按钮事件可以直接调用普通 Java ViewModel 方法，不需要一个等价的 `Q_INVOKABLE` 注解。Data Binding 生成绑定和 listener 代码，调用目标必须有可访问、签名合适的方法。

### 10.1 教学 ViewModel

```java
package com.example;

import android.view.View;
import androidx.annotation.MainThread;
import androidx.lifecycle.LiveData;
import androidx.lifecycle.MutableLiveData;
import androidx.lifecycle.Transformations;
import androidx.lifecycle.ViewModel;

public final class VoteServerViewModel extends ViewModel {
    private final MutableLiveData<Boolean> running = new MutableLiveData<>(false);
    private final LiveData<Boolean> canStart = Transformations.map(
            running, value -> !Boolean.TRUE.equals(value));

    public LiveData<Boolean> getRunning() { return running; }
    public LiveData<Boolean> getCanStart() { return canStart; }
    public LiveData<Boolean> getCanStop() { return running; }

    @MainThread
    public void startServer() {
        if (Boolean.TRUE.equals(running.getValue())) return;
        running.setValue(true); // 教学模拟
    }

    @MainThread
    public void stopServer() {
        if (!Boolean.TRUE.equals(running.getValue())) return;
        running.setValue(false);
    }

    public void onStartClicked(View view) { startServer(); }
    public void onStopClicked(View view) { stopServer(); }
}
```

`@MainThread` 用于标明调用约束，不负责切换线程。这里的按钮事件在主线程触发；真实耗时操作仍应交给 Repository/Service 的异步接口。

### 10.2 XML lambda：参数可自行适配

```xml
<layout xmlns:android="http://schemas.android.com/apk/res/android">
    <data>
        <variable name="viewModel" type="com.example.VoteServerViewModel" />
    </data>
    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical">
        <Button
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="启动服务"
            android:enabled="@{viewModel.canStart}"
            android:onClick="@{() -> viewModel.startServer()}" />
        <Button
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="停止服务"
            android:enabled="@{viewModel.canStop}"
            android:onClick="@{() -> viewModel.stopServer()}" />
    </LinearLayout>
</layout>
```

该教学工程需启用 Gradle Data Binding，并依赖 AndroidX Lifecycle。Fragment 在创建绑定并建立 View 生命周期后设置：

```java
// onViewCreated() 内；假设 binding 已由对应 layout 生成并初始化。
VoteServerViewModel viewModel = new ViewModelProvider(this)
        .get(VoteServerViewModel.class);
binding.setViewModel(viewModel);
binding.setLifecycleOwner(getViewLifecycleOwner());
```

这里另需导入 `androidx.lifecycle.ViewModelProvider`。Fragment 的绑定引用应在 `onDestroyView()` 清理。使用 LiveData 的 XML 绑定需要生命周期所有者；仅设置 ViewModel 不足以使后续 LiveData 更新正常参与界面观察。[Android：Data Binding 与架构组件](https://developer.android.com/topic/libraries/data-binding/architecture)

### 10.3 方法引用：签名要匹配 listener

可以将前面两个按钮的 `android:onClick` 分别改为：

```xml
android:onClick="@{viewModel::onStartClicked}"
android:onClick="@{viewModel::onStopClicked}"
```

`OnClickListener.onClick(View)` 接受 View 参数，所以上面的适配方法也接受 View。无参 `startServer()` 适合 XML lambda，不能原样当成同签名方法引用。lambda 可以忽略 listener 的参数，或使用 `@{(view) -> viewModel.onStartClicked(view)}` 显式传递。

这里的 Data Binding 表达式与 `android:onClick="methodName"` 查找 Activity 点击方法的旧式写法不同。[Android：方法引用与 listener 绑定](https://developer.android.com/topic/libraries/data-binding/expressions#event_handling)

```text
Qt：onClicked → 元对象公开的方法
Android：生成的 listener → 普通可访问方法

Qt：enabled 绑定 canStart + NOTIFY
Android：enabled 绑定 canStart + LiveData 观察
```

两者都需要单独表达按钮可用性，事件绑定本身不会根据方法名寻找命令守卫。

## 11. Android 补充：Compose 事件回调

Compose 不使用 XML 方法暴露机制，可以把操作作为函数参数传给组件：

```kotlin
@Composable
fun ServiceControlPanel(
    running: Boolean,
    onStartRequested: () -> Unit,
    onStopRequested: () -> Unit
) {
    Row {
        Button(enabled = !running, onClick = onStartRequested) {
            Text("启动服务")
        }
        Button(enabled = running, onClick = onStopRequested) {
            Text("停止服务")
        }
    }
}
```

此片段需导入 Compose runtime、layout 和所选 Material 组件。Route 层在已通过 Compose State 或生命周期感知的 Flow 收集获得 `state.running` 后组装：

```kotlin
ServiceControlPanel(
    running = state.running,
    onStartRequested = viewModel::startServer,
    onStopRequested = viewModel::stopServer
)
```

这里假设 Kotlin ViewModel 有两个无参方法，`state` 是可观察的页面状态；这是独立示意，不直接引用第 10 节的 Java/LiveData ViewModel。

调用函数引用不自动启动协程。ViewModel 可以使用 `viewModelScope` 调用异步 Service；阻塞工作仍需合适的调度器。异步时还应把 `busy/canStart/canStop` 纳入页面状态，不能继续只依赖一个 `running`。

这与 QML 可复用组件的“状态向下、事件向上”相近，但 Compose 重组、Android ViewModel 生命周期和 Qt QObject 机制不同。[Compose：状态提升](https://developer.android.com/develop/ui/compose/state-hoisting)

## 12. 横向对照与选择规则

| 维度 | Qt/QML | WPF ICommand | Caliburn.Micro | Android XML Data Binding | Compose |
| --- | --- | --- | --- | --- | --- |
| 业务入口 | 公有 invokable／slot | 命令调用业务方法 | Action 调用方法 | listener 调用方法 | 回调调用方法 |
| View 触发 | `onClicked`／组件 signal | `Command` 绑定 | `Message.Attach`／名称约定 | XML lambda／方法引用 | `onClick` 函数参数 |
| 方法暴露机制 | Qt 元对象登记 | 公有命令属性与绑定 | 框架查找 Action 目标及方法 | 生成绑定代码 | Kotlin 函数／函数引用 |
| 参数 | 方法参数 | `CommandParameter` | Action 参数 | lambda 参数／listener 参数 | lambda 捕获／回调参数 |
| 可执行性 | 显式 `enabled` 绑定属性 | `CanExecute` | `Can<动作名>` 守卫 | 显式 `android:enabled` 绑定 | 显式 `enabled` 状态 |
| 可执行性更新 | 属性 `NOTIFY` | `CanExecuteChanged` | 守卫属性变化通知 | 可观察属性／LiveData | 可观察状态触发重组 |
| 同步返回值 | 可直接读取支持的返回类型 | `Execute` 返回 void | 由 Action 机制处理，不当作属性绑定返回值 | 受 listener 及表达式签名约束 | 由回调类型决定，点击通常 Unit |
| 后台执行 | 宏不提供，Service/Worker 负责 | 接口不提供，异步命令实现负责 | 需要相应异步实现 | 绑定不提供 | 回调不提供 |

这些是职责对照，不是类型等价。尤其不能把 `Q_INVOKABLE` 单独等同于完整的 `ICommand`：Qt 示例的相似能力由“可调用方法 + 可用性属性 + 通知 + View 绑定”共同组成。

选择接口时：

- 需要随时读取并绑定的当前值：使用 `Q_PROPERTY`。
- 用户请求执行操作：使用语义明确的 invokable 或公有 slot。
- 组件向父页面输出用户意图：使用 QML signal。
- 后端报告完成、失败或连接变化：使用 signal，并更新需要保存的状态。
- C++ 内部调用或函数指针连接：按正常 C++ 接口设计，不必为了连接给所有方法加宏。
- 需要重复查询的计算值：优先用有明确依赖通知的属性；仅调用查询方法不等于建立完整的变化通知链。

## 13. 常见问题与排查顺序

| 现象 | 优先检查 | 处理方向 |
| --- | --- | --- |
| QML 报方法不存在或不是函数 | 方法是否公有、是否 invokable／slot、实际对象类型 | 检查元对象签名与注入对象 |
| 类型名无法识别 | import、模块 URI、QML 注册及构建来源 | 检查模块生成和加载，而非只加 invokable |
| 注册代码编译报不能继承 final 类型 | 是否把该注册路径的 C++ 类标记为 final | 本例移除 final，保留 QML_UNCREATABLE |
| 根 required 属性未初始化 | 是否在加载前传入同名初始属性 | 修正实例注入；查看引擎加载错误 |
| 对象为空或已失效 | 所有权、创建顺序、销毁时机 | 保证实例活到 QML 不再使用它 |
| 按名称 invokeMethod 返回 false | 方法名、参数数量／类型、返回类型 | 查元对象，使用准确参数类型 |
| 方法调用成功但操作被拒绝 | 业务返回值、当前服务状态、输入范围 | 区分调用成功和业务成功 |
| 点击后窗口卡住 | 方法体是否阻塞 GUI 线程 | 将耗时工作交给异步 Service/Worker |
| 队列请求未执行 | 接收对象线程、事件循环、对象是否销毁 | 修正投递上下文和生命周期 |
| 按钮可用性或文字不更新 | Getter 依赖变化后是否发对应 NOTIFY | 补齐状态及派生属性通知 |
| 重复启动或停止 | 是否只禁用了按钮，是否遗漏 busy／中间状态 | ViewModel/Service 独立校验并防重入 |
| Android 方法引用编译失败 | 方法是否接受 listener 需要的参数 | 加适配方法或改用 XML lambda |
| WPF／Caliburn 按钮状态不更新 | 命令重查事件／守卫属性通知 | 发出对应通知，而非只修改字段 |

Qt 建议依次排查：

```text
实例是否有效
    → 类型／模块是否可见
    → 方法是否公有且已登记
    → 参数与返回类型是否匹配
    → 线程和事件循环是否符合调用方式
    → 业务校验与后续状态通知是否正确
```

不要通过暴露全部内部方法来修复某一个调用错误。ViewModel 应公开稳定的界面操作，供应商对象、线程控制和 SDK 原始回调仍留在应用内部。

## 14. 速查清单

```text
普通 public 方法
    → C++ 可调用，不自动暴露为 QML 方法

public Q_INVOKABLE / public slots
    → 元对象中的可调用入口

QML_ELEMENT + qt_add_qml_module
    → 模块认识类型，不等于已经创建实例

setInitialProperties + required property
    → 根组件拿到 C++ 创建的实例

方法 + canStart/canStop 属性 + NOTIFY + enabled 绑定
    → 接近 WPF 命令的执行与可用性职责

Q_INVOKABLE
    → 不提供自动异步、自动线程切换或命令守卫

同步返回值
    → 当前调用的结果，语义由接口定义

异步请求
    → 用后续状态／完成信号反馈结果
```

## 15. 验证范围与官方参考

本文以 Qt 6.8.3 为示例基线；Qt 在线 `qt-6.8` 文档可能显示该分支较新的补丁版本。第 6 节参数推导重载从 Qt 6.5 起提供，本文所用 lambda 重载也在 6.8.3 中可用；不能直接把这些写法复制到 Qt 5.12 工程而不核对兼容 API。

2026-10-02 在 Apple Silicon macOS、Qt 6.8.3 环境中，将第 3 节四个文件提取到临时目录，在入口中加入校验代码，完成以下检查：

- CMake 配置、AUTOMOC、QML 类型注册、QML 编译与可执行文件链接通过。
- QML 根组件加载、强类型实例注入通过；直接触发按钮的 `clicked` 信号后，C++ 状态和按钮 `enabled` 绑定正确更新。
- invokable 方法登记、普通 Getter 不作为元方法登记、重复启动／停止保护、端口范围及运行中修改保护通过。
- 按名称带参调用、同步布尔和字符串返回值、QML 参数及返回值转换通过；确认“调用成功、业务拒绝”可同时成立。
- 按名称与 lambda 的队列投递通过，并使用 `sendPostedEvents(..., QEvent::MetaCall)` 定向派发目标对象的调用事件，确认请求不是立即执行。
- 文档代码围栏、本地链接及完整 Android XML 示例的 XML 结构检查通过；XML 结构检查不等于 Android Data Binding 编译验证。

验证边界：无界面程序在处理完整 GUI 事件循环时出现崩溃，未确认原因；定向派发 MetaCall 的校验程序正常退出。上述结果只覆盖加载、调用和绑定，没有验证窗口绘制、真实鼠标点击或完整 GUI 事件循环的稳定性，也没有验证跨线程 Worker 和麒麟运行行为。

WPF、Caliburn.Micro、Android 和 Compose 示例用于机制对照，不提供完整平台工程；未在对应运行时构建或运行。异步部分只展示职责和接口，没有真实 SDK、网络或 Worker 实现。

官方参考：

- [Qt：Q_INVOKABLE 宏](https://doc.qt.io/qt-6.8/qobject.html#Q_INVOKABLE)
- [Qt：向 QML 暴露 C++ 方法](https://doc.qt.io/qt-6.8/qtqml-cppintegration-exposecppattributes.html#exposing-methods-including-qt-slots)
- [Qt：QML 与 C++ 类型转换及所有权](https://doc.qt.io/qt-6.8/qtqml-cppintegration-data.html)
- [Qt：QMetaObject](https://doc.qt.io/qt-6.8/qmetaobject.html)
- [Qt：线程与 QObject](https://doc.qt.io/qt-6.8/threads-qobject.html)
- [WPF：命令概览](https://learn.microsoft.com/dotnet/desktop/wpf/advanced/commanding-overview)
- [Caliburn.Micro：Actions](https://caliburnmicro.com/documentation/actions)
- [Android：绑定表达式与事件处理](https://developer.android.com/topic/libraries/data-binding/expressions)
- [Android：Data Binding 与架构组件](https://developer.android.com/topic/libraries/data-binding/architecture)
- [Compose：状态提升](https://developer.android.com/develop/ui/compose/state-hoisting)
