# Qt QTimer：基础用法、事件循环与网络超时实战

## 1. 文档目标与适用版本

本文结合 `qt_daliy_picture` 每日一图项目，讲解 `QTimer` 的常用 API、异步执行机制、生命周期、线程归属，以及网络请求取消和超时检查的关系。

项目示例以 Qt 6.8.3、C++17 为基准，代码核对日期为 2026-10-15。标注为“可选写法”的代码仅用于学习，不表示每日一图项目已经采用该实现。

重点解决以下问题：

- `QTimer` 是否会创建线程，是否像 `sleep()` 一样阻塞？
- `singleShot(0, ...)` 为什么能安排首次刷新？
- `singleShot(30000, reply, [reply] {...})` 中两个 `reply` 有什么区别？
- 连续点击刷新时，旧请求和旧超时检查分别如何处理？
- 如何显式停止计时，或者实现重新计时、防抖与周期任务？
- “30 秒”能否保证绝不提前、恰好到点执行？

项目中的两处业务调用：

| 位置 | 调用 | 目的 |
| --- | --- | --- |
| `src/main.cpp` | `QTimer::singleShot(0, &viewModel, &DailyPictureViewModel::refresh)` | 事件循环启动后安排首次刷新 |
| `src/DailyPictureViewModel.cpp` | `QTimer::singleShot(30000, reply, [reply] {...})` | 给当前 HTTP 请求安排总时限检查 |

测试代码还使用了一次 `singleShot()`，按预设延迟发出模拟响应，用于验证异步请求和取消行为。

## 2. QTimer 是什么

`QTimer` 是继承自 `QObject` 的 Qt Core 定时器类。它负责安排任务的执行时机，支持重复触发和单次触发；它本身不负责网络请求，也不直接实现“取消请求”。

```cpp
#include <QTimer>
```

独立使用 Qt Core 时，CMake 配置为：

```cmake
find_package(Qt6 REQUIRED COMPONENTS Core)
target_link_libraries(my_target PRIVATE Qt6::Core)
```

每日一图已经链接 Qt Gui、Network 和 Quick，Qt Core 会通过依赖关系引入。

### 2.1 两种常见使用形式

| 形式 | 典型用途 | 是否持有可控制的对象 |
| --- | --- | --- |
| 创建 `QTimer`，连接 `timeout()` 后 `start()` | 周期任务、需要停止或重新计时的任务 | 是 |
| 静态 `QTimer::singleShot(...)` | 延迟执行一次、零毫秒延后执行 | 通常不向调用者提供定时器句柄 |

静态 `singleShot()` 返回 `void`，不需要调用者手动创建或删除内部定时器对象。

## 3. 定时器与事件循环

### 3.1 注册任务后，当前代码继续执行

```cpp
// 假设这段代码位于 QObject 派生类的方法中。
QTimer::singleShot(1000, this, [] {
    qDebug() << "执行延迟任务";
});

qDebug() << "注册任务后立即继续执行";
```

第二条日志先执行。第一条日志要等所属线程的事件循环稍后调度。

```text
注册定时任务
    ↓
当前函数继续运行并返回
    ↓
事件循环处理鼠标、绘制、网络等事件
    ↓
达到触发条件，线程可以处理事件
    ↓
执行定时回调
```

GUI 应用通常在 `app.exec()` 中进入主事件循环。控制台 Qt 应用同样可以通过 `QCoreApplication::exec()` 运行事件循环。没有事件循环，依赖它的定时回调就无法正常执行。

参考：[Qt 定时器机制](https://doc.qt.io/qt-6.8/timers.html)、[QCoreApplication::exec()](https://doc.qt.io/qt-6.8/qcoreapplication.html#exec)。

### 3.2 与 sleep 的区别

| 行为 | 主线程中等待时的效果 |
| --- | --- |
| `sleep()` / `QThread::sleep()` | 当前线程被阻塞，无法正常处理界面与事件 |
| `QTimer::singleShot()` | 注册后返回，允许事件循环继续处理其他任务 |
| 定时回调中执行长时间计算 | 回调仍会占住执行它的线程 |

因此，使用定时器不会自动把重计算、图片解码或磁盘操作移到后台线程。

### 3.3 到时间不等于立即抢占执行

如果主线程正在同步执行一个耗时任务，定时器不能打断这段代码。回调要等线程重新有机会处理事件。

例如，安排 1 秒后的回调，再同步计算 5 秒，回调可能在计算结束后才运行。

“异步安排”和“并行执行”是两个不同概念。项目里的定时器并没有为每个 HTTP 请求额外创建一个线程。

## 4. QTimer 对象的基本用法

下面的代码片段假定在 QObject 派生类的方法中执行。

### 4.1 重复定时器

```cpp
auto *timer = new QTimer(this);

connect(timer, &QTimer::timeout, this, [] {
    qDebug() << "执行一次周期任务";
});

timer->start(1000);
```

默认 `singleShot` 属性为 false，因此会按设定间隔重复发出 `timeout()`。父对象 `this` 销毁时会清理这个定时器。

### 4.2 一次性定时器对象

```cpp
auto *timer = new QTimer(this);
timer->setSingleShot(true);

connect(timer, &QTimer::timeout, this, [] {
    qDebug() << "只执行一次";
});

timer->start(2000);
```

单次触发后定时器停止计时，但这个 `QTimer` 对象不会因此自动删除。可以重新调用 `start()` 使用它，也可以由父对象在销毁时清理。

### 4.3 常用 API

| API | 含义 |
| --- | --- |
| `setInterval(1000)` | 设置间隔，整数单位为毫秒 |
| `start()` | 使用已设置的间隔启动或重新启动 |
| `start(1000)` | 设置间隔并启动或重新启动 |
| `stop()` | 停止之后的计时触发 |
| `setSingleShot(true)` | 设置为单次触发 |
| `isSingleShot()` | 查询是否为单次触发模式 |
| `isActive()` | 查询是否正在计时 |
| `remainingTime()` | 查询剩余毫秒数；未运行时为 -1，已到期等待处理时为 0 |
| `setTimerType(...)` | 选择定时器精度类型 |
| `timeout()` | 到期时发出的信号 |

`stop()` 不会中断一个已经开始运行的回调；回调已经启动的业务操作也需要自己的取消机制。

参考：[QTimer 类文档](https://doc.qt.io/qt-6.8/qtimer.html)。

### 4.4 start 会重新计时

```cpp
timer->start(1000);
// 过了约 600 毫秒后再次执行：
timer->start(1000);
```

第二次调用会重新开始一个 1000 毫秒的计时周期，不会在同一个 QTimer 对象中叠加另一个独立计时任务。

对运行中的定时器改变 interval，也会影响当前计时，不能把 interval 当成只在下一轮生效的普通配置字段。

## 5. singleShot 的参数和回调写法

### 5.1 推荐理解的调用形式

```cpp
QTimer::singleShot(延迟时间, 上下文对象, 回调函数);
```

带精度参数的形式为：

```cpp
QTimer::singleShot(延迟时间, 定时器类型, 上下文对象, 回调函数);
```

| 参数 | 作用 |
| --- | --- |
| 延迟时间 | 整数按毫秒解释，也可以传支持的 `std::chrono` 时长 |
| 上下文对象 | 关联回调的生命周期，并指定回调运行的线程 |
| 回调函数 | lambda、成员函数指针等可调用对象 |

### 5.2 成员函数指针

```cpp
QTimer::singleShot(1000, this, &SomeObject::updateDisplay);
```

`&SomeObject::updateDisplay` 表示“要调用哪个成员函数”，此刻并没有调用它。到期后，在 `this` 指向的对象上执行该函数。

这种类型安全写法不要求目标方法一定声明在 `slots:` 下；Qt 槽也是普通 C++ 成员函数，二者不应混淆。

### 5.3 lambda

```cpp
const QString message = QStringLiteral("延迟显示的内容");

QTimer::singleShot(1000, this, [message] {
    qDebug() << message;
});
```

`[message]` 按值保存字符串，原函数返回后回调仍能使用自己的副本。

不要引用捕获一个会在回调执行前离开作用域的局部变量：

```cpp
// 错误示例：局部变量离开作用域后，回调里的引用失效。
QString message = QStringLiteral("临时内容");
QTimer::singleShot(1000, this, [&message] {
    qDebug() << message;
});
```

上下文对象只保护与它关联的回调，不会自动让所有被捕获变量延长生命周期。

### 5.4 多次 singleShot 会产生多个任务

```cpp
QTimer::singleShot(1000, this, [] { qDebug() << "任务 A"; });
QTimer::singleShot(1000, this, [] { qDebug() << "任务 B"; });
```

两次调用彼此独立，不会因为上下文相同就合并、去重或重置彼此的计时。不要用这种写法直接替代需要“重置等待时间”的防抖定时器。

## 6. 上下文对象、指针捕获与生命周期

### 6.1 第二个参数与捕获列表不是一回事

```cpp
QTimer::singleShot(30000, reply, [reply] {
    reply->abort();
});
```

| 位置 | 含义 |
| --- | --- |
| 第二个参数 `reply` | 上下文对象；它提前销毁后，回调不再调用 |
| 捕获列表 `[reply]` | 按值保存指针地址，供回调访问对象 |

按值捕获指针不等于复制对象，也不会增加该 QObject 的所有权。

公开 API 保证的是：上下文销毁后不会再调用回调。不能仅据此断言内部定时器已在那一瞬间执行 `stop()` 或被物理释放；不要依赖未公开的实现细节。

### 6.2 没有上下文时的风险

```cpp
// 风险示例：reply 可能在这 30 秒内提前销毁。
QTimer::singleShot(30000, [reply] {
    reply->abort();
});
```

这段代码只保存了裸指针，没有把回调生命周期关联到响应对象。网络响应提前销毁后，稍后访问指针会出问题。

当前项目的带上下文形式可以避免这一问题。

### 6.3 QPointer 的作用

```cpp
QPointer<QNetworkReply> active;
active = reply;
```

`QPointer` 观察 QObject，目标销毁后自动变空，但不负责删除目标对象。

```cpp
active.clear();
```

只清空观察指针，不取消请求，不删除响应，也不会停止已经安排的定时任务。

### 6.4 QTimer 本身也需要正确的生命周期

```cpp
void SomeObject::begin()
{
    QTimer timer;
    timer.start(1000);
} // timer 销毁，不能指望它继续触发。
```

通常可以把定时器做成成员，或者通过 `new QTimer(this)` 使用 QObject 父子关系管理。只有局部对象一直活到事件循环处理它时，局部定时器才可能正常触发；问题不是“栈对象不能用”，而是它是否活得足够久。

## 7. 项目实战一：零毫秒首次刷新

项目入口中的代码：

```cpp
QTimer::singleShot(0, &viewModel, &DailyPictureViewModel::refresh);
return app.exec();
```

### 7.1 为什么使用 0

零毫秒表示尽快交给事件循环调度，不是在 `singleShot()` 这条语句中直接同步执行。

```text
创建 ViewModel
    ↓
加载 QML，创建根对象
    ↓
安排零毫秒首次刷新
    ↓
app.exec() 进入事件循环
    ↓
事件循环有机会时执行 refresh()
```

它不保证界面已完成第一帧绘制，也不保证优先于其他网络、绘制或输入事件。

### 7.2 能不能直接调用 refresh

可以写成：

```cpp
viewModel.refresh();
return app.exec();
```

由于网络请求本身异步，直接调用不一定有问题。当前写法明确将首次业务动作交给事件循环调度，使启动阶段与后续按钮触发的执行环境更一致。

### 7.3 不要混淆单次零定时器与重复零定时器

```cpp
QTimer::singleShot(0, this, [] {
    // 只安排一次。
});
```

和下面的重复定时器不同：

```cpp
timer->start(0);
```

若 timer 未设置单次模式，会尽可能频繁触发。每次回调都应尽快返回，不要用持续的零间隔循环占满事件循环。

## 8. 项目实战二：HTTP 请求总时限

项目当前代码：

```cpp
void DailyPictureViewModel::track(QNetworkReply *reply)
{
    m_active = reply;

    QTimer::singleShot(30000, reply, [reply] {
        if (!reply->isFinished()) {
            reply->setProperty("deadlineExceeded", true);
            reply->abort();
        }
    });
}
```

### 8.1 m_active 跟踪当前请求

`m_active = reply` 让下一轮刷新能够找到当前请求并取消。JSON 请求完成后进入图片请求阶段，跟踪对象会随之切换到图片响应。

### 8.2 定时器只检查创建它时的那个请求

lambda 捕获的是本次 `reply` 指针，不是以后可能指向其他请求的 `m_active`。

```text
请求 A → track(A) → 超时检查 A
请求 B → track(B) → 超时检查 B
```

因此，旧检查不会因为 `m_active` 改成 B，就误取消新请求 B。

### 8.3 isFinished 处理请求已结束的情况

`finished` 包括成功、失败和主动取消，不只表示成功。

| 到检查时的状态 | 行为 |
| --- | --- |
| 响应对象已销毁 | 上下文失效，不调用 lambda |
| 响应已结束但还存活 | `isFinished()` 为 true，不再次取消 |
| 请求尚未结束 | 添加超时标记并取消 |

### 8.4 deadlineExceeded 是项目自定义标记

```cpp
reply->setProperty("deadlineExceeded", true);
reply->abort();
```

这是 QObject 的动态属性，不是 Qt 内置的 HTTP 超时字段。项目借此区分“总时限触发的取消”和其他取消原因。

必须先设置标记，再调用 `abort()`。因为取消可能同步触发完成回调，若先取消、后标记，错误处理就可能读不到原因。

### 8.5 两种超时互相补充

项目还配置了：

```cpp
m_network->setTransferTimeout(10000);
```

| 机制 | 解决的问题 |
| --- | --- |
| 10 秒传输超时 | 连续一段时间没有数据交换 |
| 约 30 秒总时限检查 | 即使零星有数据，请求也不应无限持续 |

JSON 请求和图片请求分别调用 `track()`，分别计时。当前不是“从点击刷新到图片显示，整个流程总共只能 30 秒”。

网络等待异步，但回调中的图片解码和磁盘写入仍可能占用主线程；总时限回调不能抢占这些同步操作。

参考：[传输超时](https://doc.qt.io/qt-6.8/qnetworkrequest.html#setTransferTimeout-1)。

## 9. 连续点击刷新：取消请求、停止计时、使回调失效

### 9.1 刷新入口主动取消旧请求

```cpp
const quint64 generation = ++m_generation;

if (m_active) {
    const auto oldReply = m_active;
    m_active.clear();
    oldReply->abort();
}
```

真正取消客户端请求的是 `abort()`。它不保证服务端已经开始执行的工作也会被撤销。

### 9.2 为什么先增加编号

每个完成回调都保存发起那轮请求时的编号：

```cpp
if (generation != m_generation)
    return;
```

```text
旧请求 A 的编号为 1
    ↓
用户再次刷新，当前编号变为 2
    ↓
abort(A) 可能立即触发旧完成回调
    ↓
旧回调比较 1 和 2，直接返回
```

如果先取消、后增加编号，旧回调可能误以为自己仍是最新请求。

### 9.3 旧超时检查如何处理

项目启用了：

```cpp
m_network->setAutoDeleteReplies(true);
```

旧请求结束后，响应对象会安排自动清理。对象销毁后，与它关联的 `singleShot()` 回调不会再调用。若对象仍存活而检查先执行，`isFinished()` 也会阻止再次取消。

这里没有显式调用定时器的 `stop()`，因为静态调用没有把可控制的 QTimer 对象交给调用方。

| 对象或操作 | 连续刷新时的处理 |
| --- | --- |
| 旧网络请求 | 显式 `abort()` |
| 旧响应对象 | 请求结束后安排自动清理 |
| 旧超时回调 | 上下文销毁后失效，或因已完成而不执行取消 |
| 旧业务结果 | 刷新编号过期，不更新界面 |
| `m_active.clear()` | 只清空当前跟踪指针 |

“取消请求”“显式停止计时”和“回调失效”是不同动作。当前实现组合了请求取消、回调生命周期保护和结果编号检查。

## 10. 可选写法：显式持有并停止超时定时器

如果希望请求结束时明确停止计时，可以使用下面的形式替换 `track()` 内的静态计时部分。这个例子没有修改每日一图现有实现。

```cpp
void DailyPictureViewModel::track(QNetworkReply *reply)
{
    m_active = reply;

    auto *timeoutTimer = new QTimer(reply);
    timeoutTimer->setSingleShot(true);
    timeoutTimer->setTimerType(Qt::PreciseTimer);

    connect(timeoutTimer, &QTimer::timeout, reply, [reply] {
        if (!reply->isFinished()) {
            reply->setProperty("deadlineExceeded", true);
            reply->abort();
        }
    });

    connect(reply, &QNetworkReply::finished,
            timeoutTimer, &QTimer::stop);

    timeoutTimer->start(30000);
}
```

执行关系：

```text
请求正常完成或取消
    ↓
finished 信号
    ↓
timeoutTimer->stop()
    ↓
响应对象随后销毁
    ↓
其子对象 timeoutTimer 也被销毁
```

如果请求没有结束，定时器到期后执行取消。

| 对比 | 静态 singleShot | 持有 QTimer 对象 |
| --- | --- | --- |
| 代码量 | 少 | 较多 |
| 直接调用 stop | 无直接句柄 | 可以 |
| 查询剩余时间 | 无直接句柄 | 可以 |
| 重新开始同一个计时 | 需另行设计 | 可以调用 start |
| 生命周期 | 上下文保护回调 | 父子关系加信号连接 |

## 11. 定时精度与 30 秒的准确含义

### 11.1 三种常见类型

| 类型 | 提前触发特征 | 使用场景 |
| --- | --- | --- |
| `Qt::PreciseTimer` | 不提前触发 | 对最早执行时间敏感的任务 |
| `Qt::CoarseTimer` | 允许在间隔 5% 的范围内提前 | 普通定时任务 |
| `Qt::VeryCoarseTimer` | 允许最多约 500 毫秒提前 | 对精度要求较低的任务 |

所有类型都可能因为调度繁忙或事件循环阻塞而延后。PreciseTimer 不等于硬实时计时，也不保证实际精度总能达到 1 毫秒。

参考：[Qt::TimerType](https://doc.qt.io/qt-6.8/qt.html#TimerType-enum)。

### 11.2 对象默认值与静态方法默认策略要分开

QTimer 对象的 `timerType` 默认值为 `Qt::CoarseTimer`。

对于未显式传类型的静态 `singleShot()`，本机 Qt 6.8.3 的 `qtimer.h` 模板实现按间隔选择：

| 间隔 | 本机 Qt 6.8.3 的默认选择 |
| --- | --- |
| 小于 2 秒 | `Qt::PreciseTimer` |
| 大于等于 2 秒 | `Qt::CoarseTimer` |

该策略来自版本实现核对，不应被当成所有 Qt 版本永远不变的 API 承诺。核对位置为 Qt 6.8.3 头文件 `QtCore/qtimer.h` 中的 `defaultTypeFor()`。

因此，当前的 `singleShot(30000, reply, ...)` 使用粗略定时器。5% 的提前误差范围对应 1.5 秒，但不表示每次都会提前这么多。

把这段代码描述成“约 30 秒检查”更准确，不能仅靠现有写法断言“一定超过 30 秒才取消”。

### 11.3 明确要求不早于 30 秒时

```cpp
QTimer::singleShot(30000, Qt::PreciseTimer, reply, [reply] {
    if (!reply->isFinished()) {
        reply->setProperty("deadlineExceeded", true);
        reply->abort();
    }
});
```

若采用这种实现，应同步检查相关注释和错误提示，保证业务表述与计时策略一致。

### 11.4 重复定时器延误不会无限补发

当事件循环处理不过来、多个触发时刻都已经过去时，Qt 不会为每个错过的周期无限补发 `timeout()`；会处理一次超时通知，再恢复计时。

如果要显示真实经过时间，应使用时间戳或 `QElapsedTimer` 测量，不要简单地将“timeout 次数 × 间隔”当成准确耗时。

## 12. 使用 chrono 表达时间

```cpp
#include <chrono>

using namespace std::chrono_literals;

QTimer::singleShot(30s, reply, [reply] {
    // 延迟约 30 秒执行。
});

timer->start(500ms);
```

这种写法把单位直接写在值上，比裸整数更容易阅读。传入 `30s` 不会自动把默认定时器类型改成 PreciseTimer；是否显式指定精度是另一项决定。

需要明确精度时：

```cpp
QTimer::singleShot(30s, Qt::PreciseTimer, reply, [reply] {
    // 不早于目标时间，再由事件循环调度。
});
```

虽然某些接口可接收细粒度 chrono 时长，但这不等于平台和事件循环能提供相同精度。Qt 版本之间也存在重载差异，应以目标版本为准。

QTimer 的常规间隔受 int 毫秒范围约束，最大正间隔约为 24.8 天。不要依赖负间隔的行为，也不要直接传入溢出的超大毫秒值。需要更长时间范围时，可研究 Qt 6.8 引入的 `QChronoTimer`。

## 13. 线程归属

### 13.1 当前项目在主线程处理定时回调

ViewModel 和网络响应在主线程中使用，传入它们作为上下文的定时回调也在相应线程中执行。

`singleShot()` 不会因为参数是 30 秒，就创建一个线程睡眠 30 秒后直接操作对象。

### 13.2 工作线程也可以使用定时器

前提是该线程有运行中的 Qt 事件循环。通常应在目标线程内创建或正确迁移工作对象，并在对象所属线程中创建、启动和停止定时器。

普通 QTimer 对象的 `start()`、`stop()` 不应从不相关线程直接调用。需要跨线程控制时，可以用带正确接收对象的排队信号槽，将操作投递到定时器所属线程。

带上下文的静态 `singleShot()` 回调会在上下文所属线程执行。这不等于其中捕获的其他 QObject 都可以跨线程访问；网络对象仍需遵守自己的线程归属规则。

## 14. 常用扩展：防抖与周期任务

### 14.1 防抖需要重启同一个定时器

防抖是“最后一次输入之后，等待一段时间再执行”。下面是独立的教学示例，不是项目当前的刷新实现。

```cpp
class DebouncedSearch final : public QObject
{
public:
    explicit DebouncedSearch(QObject *parent = nullptr)
        : QObject(parent), m_timer(new QTimer(this))
    {
        m_timer->setSingleShot(true);
        connect(m_timer, &QTimer::timeout, this, [this] {
            qDebug() << "查询最新输入：" << m_keyword;
        });
    }

    void setKeyword(const QString &keyword)
    {
        m_keyword = keyword;
        m_timer->start(300);
    }

private:
    QTimer *m_timer;
    QString m_keyword;
};
```

连续输入会反复重置同一个 300 毫秒计时器，直到输入停止才触发。示例需包含 `QObject`、`QTimer`、`QString` 和 `QDebug` 头文件。

每日一图的连续刷新属于“立即发起新一轮，并取消旧请求”，不是防抖；不要混淆这两种产品行为。

### 14.2 周期刷新要考虑任务重叠

```cpp
auto *refreshTimer = new QTimer(this);

connect(refreshTimer, &QTimer::timeout, this, [this] {
    if (!m_loading)
        refresh();
});

refreshTimer->start(60000);
```

如果在 ViewModel 中采用这个可选示例，表示约每分钟尝试刷新一次，已有请求时跳过。周期触发与异步请求耗时是两回事，应明确选择跳过、排队还是替换旧请求，而不是默认认为上次任务一定已经结束。

## 15. QTimer 与 QML Timer 等工具的区别

| 工具 | 用途 |
| --- | --- |
| C++ `QTimer` | C++ 业务定时、超时检查、周期任务 |
| QML `Timer` | QML 层的时间驱动行为 |
| `QElapsedTimer` | 测量经过时间，不负责安排回调 |
| `QChronoTimer` | Qt 6.8 引入，更宽的时长范围和细粒度时长表示 |
| `QBasicTimer` | 更底层的轻量计时封装，通常配合 timerEvent |

QML Timer 示例：

```qml
import QtQuick

Item {
    Timer {
        interval: 1000
        repeat: true
        running: true
        onTriggered: console.log("执行界面定时任务")
    }
}
```

QML Timer 与动画计时机制同步，受界面时钟粒度影响，不应因为 interval 单位为毫秒，就把它当成毫秒级硬实时工具。当前项目将 HTTP 生命周期和超时放在 ViewModel 中，使界面只负责展示与触发命令。

参考：[QML Timer](https://doc.qt.io/qt-6.8/qml-qtqml-timer.html)、[QElapsedTimer](https://doc.qt.io/qt-6.8/qelapsedtimer.html)、[QChronoTimer](https://doc.qt.io/qt-6.8/qchronotimer.html)。

## 16. 项目测试如何验证相关行为

`tests/DailyPictureViewModelTest.cpp` 的 FakeReply 使用：

```cpp
QTimer::singleShot(m_response.delayMs, this, [this] {
    if (isFinished())
        return;

    // 设置模拟响应状态和内容。
    setFinished(true);
    emit readyRead();
    emit finished();
});
```

这样测试也遵守异步时序，不是在发起请求的同一个调用栈里立刻完成。

主要验证两个取消场景：

- 取消立即触发结束回调：检查刷新编号是否已经先失效。
- 模拟旧响应取消后仍延迟到达：检查旧结果是否会覆盖新状态。

第二种是主动构造的测试边界，不表示真实 QNetworkReply 的 abort 一定会继续传输；它用于证明结果编号保护自身有效。

`QTRY_VERIFY()`、`QTRY_COMPARE()` 会在等待期间处理事件，适合等待异步条件。相关测试验证了刷新和取消流程，但现有测试没有验证真实的 30 秒总时限精度，不能从测试通过推断计时截止点已被验证。

## 17. 常见误区速查

| 误区 | 正确理解 |
| --- | --- |
| QTimer 会创建后台线程 | 它主要通过事件循环安排任务 |
| singleShot(0) 就是立即调用 | 它把任务交给事件循环稍后调度 |
| 零毫秒刷新保证窗口第一帧已绘制 | 不保证绘制与其他事件的先后顺序 |
| 捕获 reply 指针会让对象继续存活 | 指针复制不增加 QObject 所有权 |
| 上下文销毁必然等于内部定时器立即 stop | API 保证回调不再调用，不应推断内部资源的瞬时状态 |
| m_active.clear() 取消请求 | 它只清空跟踪指针，取消需要 abort |
| 调用 abort 后旧回调绝不会执行 | 取消本身可能触发完成回调，需要检查刷新编号 |
| 旧超时检查会取消新的 m_active | 当前回调捕获的是旧 reply，不会自动改为新请求 |
| 多次 singleShot 相互重置 | 它们通常是多个独立任务 |
| QTimer 单次触发后对象自动销毁 | 普通 QTimer 对象只是停止，仍需管理生命周期 |
| 定时器 stop 能打断正在执行的回调 | 已开始的回调及业务操作需要自己的取消逻辑 |
| 30 秒定时器恰好在第 30 秒执行 | 精度类型与调度都会影响实际时间 |
| PreciseTimer 保证硬实时 | 它不提前，但仍可能延迟 |
| 使用 chrono 就会提高实际计时精度 | chrono 主要改善时长表达，实际精度另有约束 |
| 网络异步意味着图片解码和保存也都在后台 | 当前回调中的同步操作仍占用执行它们的线程 |

## 18. 官方参考与阅读建议

先理解第 3 节的事件循环，再读第 7～9 节的项目时序；需要改造可停止的超时检查时，阅读第 10～11 节。

- [QTimer：Qt 6.8 API](https://doc.qt.io/qt-6.8/qtimer.html)
- [Timers：事件循环与计时机制](https://doc.qt.io/qt-6.8/timers.html)
- [QCoreApplication::exec()](https://doc.qt.io/qt-6.8/qcoreapplication.html#exec)
- [Qt::TimerType：精度类型](https://doc.qt.io/qt-6.8/qt.html#TimerType-enum)
- [QNetworkAccessManager：自动清理响应](https://doc.qt.io/qt-6.8/qnetworkaccessmanager.html#setAutoDeleteReplies)
- [QNetworkReply::abort()](https://doc.qt.io/qt-6.8/qnetworkreply.html#abort)
- [QNetworkRequest：传输超时](https://doc.qt.io/qt-6.8/qnetworkrequest.html#setTransferTimeout-1)
- [QPointer：QObject 观察指针](https://doc.qt.io/qt-6.8/qpointer.html)
- [QElapsedTimer：耗时测量](https://doc.qt.io/qt-6.8/qelapsedtimer.html)
- [QChronoTimer](https://doc.qt.io/qt-6.8/qchronotimer.html)
- [QML Timer](https://doc.qt.io/qt-6.8/qml-qtqml-timer.html)

文中的零毫秒和 30 秒业务示例来自 `qt_daliy_picture` 的代码；精度默认策略由本机 Qt 6.8.3 头文件核对。后续升级 Qt 时，应重新核对版本相关行为，而不是把实现细节当成长期不变的约定。
