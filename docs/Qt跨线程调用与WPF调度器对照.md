# 从 WPF Dispatcher 到 Qt QMetaObject::invokeMethod

自动保存失败后，后台代码需要把错误提示交给界面。WPF 通常通过 UI 对象的 `Dispatcher` 投递回调。Qt 可以用 `QMetaObject::invokeMethod`，由一个 `QObject` 指定目标线程，再用连接类型决定是立即执行、排队，还是排队后等待。

## 从自动保存失败说起

```cpp
QMetaObject::invokeMethod(context, [context, path, reason] {
    const auto message = path.isEmpty()
        ? QStringLiteral("修改已应用，但自动保存失败：%1").arg(reason)
        : QStringLiteral("修改已应用，但自动保存失败：%1（%2）。可在当前会议使用“保存会议”重试。")
              .arg(path, reason);
    context->publish(NotificationLevel::Failure, message);
}, Qt::QueuedConnection);
```

调用方将 lambda 投递给 `context` 所属线程，然后继续执行。该线程的事件循环处理这次投递时，lambda 才会拼出提示并调用 `publish()`。如果 `context->thread()` 是 UI 线程，这一步就发生在 UI 线程。如果它属于工作线程，回调就在那里执行。

`[context, path, reason]` 按值捕获这三个变量。回调可以使用投递时保存的路径和错误原因，即使原来的局部变量已经离开作用域。捕获 `context` 复制的是指针值，对象的生命周期仍由原有的所有权机制管理。

## 与 WPF Dispatcher 对照

| Qt 连接类型 | 回调在哪执行 | 调用方何时返回 | WPF 类比 |
| --- | --- | --- | --- |
| `Qt::QueuedConnection` | `context` 所属线程处理队列时 | 投递后返回 | `Dispatcher.BeginInvoke`、`Dispatcher.InvokeAsync` |
| `Qt::BlockingQueuedConnection` | `context` 所属线程处理队列时 | 等回调执行完再返回 | 跨线程调用 `Dispatcher.Invoke` |
| `Qt::DirectConnection` | 当前调用线程 | 执行完立即返回 | 直接调用委托 |
| `Qt::AutoConnection` | 同线程就直接执行，跨线程就投递到 `context` 所属线程 | 直接执行时等回调结束，排队时投递后返回 | 可用 `CheckAccess()` 选择直接调用或 `BeginInvoke` |

在 WPF 中，可以通过界面对象的 `Dispatcher` 写出与开头相同的通知逻辑：

```csharp
void ReportAutoSaveFailure(string path, string reason)
{
    notificationView.Dispatcher.BeginInvoke(new Action(() =>
    {
        var message = string.IsNullOrEmpty(path)
            ? $"修改已应用，但自动保存失败：{reason}"
            : $"修改已应用，但自动保存失败：{path}（{reason}）。可在当前会议使用“保存会议”重试。";

        Publish(NotificationLevel.Failure, message);
    }));
}
```

两段异步代码即使从 UI 线程发起，也会排队。如果把 WPF 示例改成 `Dispatcher.Invoke`，调用方会等通知发布代码执行完。Qt 的 `BlockingQueuedConnection` 只能用于跨线程场景。同线程使用会死锁，因为这个线程一边等回调结束，一边又需要处理队列才能执行回调。跨线程时，如果目标线程也在等调用方，双方同样可能卡住。

## 传 lambda、按名称调用，以及返回值

lambda 适合就地描述一次调用，比如开头的通知。如果接收对象已经有槽，也可以把槽名交给 Qt 的元对象系统。假设 `context` 声明了 `void showAutoSaveFailure(const QString &reason)` 槽：

```cpp
const bool posted = QMetaObject::invokeMethod(
    context, "showAutoSaveFailure", Qt::QueuedConnection,
    Q_ARG(QString, reason));
```

按名称调用需要匹配元对象系统里登记的方法和参数类型。除了槽，也可以调用用 `Q_INVOKABLE` 标记的方法。Qt 6.5 起，按名称调用的模板重载可以直接传实参。Qt 6.7 起，新增的函数对象重载也支持在调用处直接传参数。开头的写法通过捕获传递数据，lambda 本身没有形参。

`invokeMethod` 返回的 `bool` 表示 Qt 是否成功发起调用。排队时，`true` 只说明投递成功。WPF 的 `BeginInvoke` 和 `InvokeAsync` 返回 `DispatcherOperation`，可以用它查看操作状态。需要异步等待执行结束时，可以写 `await notificationView.Dispatcher.InvokeAsync(() => Publish(NotificationLevel.Failure, message));`。

## Qt 里还有哪些类似写法

### 一次性投递用 invokeMethod

当你已经拿到接收对象，只需要把一项工作交给它的线程时，可以直接投递：

```cpp
QMetaObject::invokeMethod(context, [context, message] {
    context->publish(NotificationLevel::Failure, message);
}, Qt::QueuedConnection);
```

### 持续转发用排队式信号槽

如果保存模块会反复报告失败，可以在初始化时建立一次连接。之后工作线程每次发出 `autoSaveFailed`，Qt 都会把对应回调排入 `context` 所属线程的队列。

```cpp
QObject::connect(worker, &SaveWorker::autoSaveFailed, context,
    [context](const QString &message) {
        context->publish(NotificationLevel::Failure, message);
    },
    Qt::QueuedConnection);
```

### 延时或零延时调度用 QTimer::singleShot

需要延时更新界面时，把延时和上下文一起交给定时器。把下面的 `250` 改为 `0`，即可让回调尽快由事件循环处理。零延时定时器和其他事件之间的处理顺序没有保证。

```cpp
QTimer::singleShot(250, context, [context, message] {
    context->publish(NotificationLevel::Failure, message);
});
```

## 排队调用的三个边界

- **事件循环需要处理投递。** 如果目标线程始终没有运行事件循环，排队的调用就不会执行。如果 UI 线程正在做耗时工作，提示要等到它能处理队列时才会出现。
- **上下文需要在投递时有效。** 如果 `context` 在处理前销毁，Qt 会移除发给它的待处理事件。带上下文的 `QTimer::singleShot` 也只会在上下文仍存在时调用回调。回调如果还用到其他对象，需要单独考虑那些对象的生命周期。
- **局部变量优先按值捕获。** 使用 `[&]` 捕获局部的 `path` 或 `reason` 时，函数可能在回调执行前就已返回，届时引用会悬空。

## 官方参考

- [Qt 6 QMetaObject 文档](https://doc.qt.io/qt-6/qmetaobject.html)
- [Qt 6 QObject 文档](https://doc.qt.io/qt-6/qobject.html)
- [Qt 6 QTimer 文档](https://doc.qt.io/qt-6/qtimer.html)
- [Microsoft Learn 上的 WPF 线程模型](https://learn.microsoft.com/en-us/dotnet/desktop/wpf/advanced/threading-model)
