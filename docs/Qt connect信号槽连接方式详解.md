# Qt connect：信号槽连接方式、队列调用与跨线程通信详解

## 1. 文档目标与适用版本

本文以 Qt 6.8、C++17 为基准，面向刚开始学习 Qt 的 C++ 开发者。既讲“`connect` 可以怎样写”，也讲“连接之后，槽究竟在哪个线程、什么时候执行”。Qt 5 的重要兼容差异会单独指出。

重点回答：

- 函数指针、Lambda、`SIGNAL()` / `SLOT()` 分别怎么用？
- `AutoConnection`、`DirectConnection`、`QueuedConnection`、`BlockingQueuedConnection` 有什么区别？
- `UniqueConnection`、`SingleShotConnection` 如何组合，是否适用于 Lambda？
- 同线程为什么还需要队列连接？异步调用为什么仍然可能卡界面？
- `this` 作为 context 有什么用？对象销毁和 `disconnect()` 能取消哪些调用？
- 如何把工作安排到后台线程，再把结果送回主线程？

第 12 节包含三个独立完整程序和通用 CMake 配置，其他代码均为局部示例，需放到实际类或函数中。完整程序使用 Qt Core，不需要 Widgets 或 QML。

相关基础：[Qt 元对象系统与 QML 类型注册](Qt元对象系统与QML类型注册.md)、[Qt 元对象注解宏详解](Qt%20元对象注解宏详解.md)。

## 2. connect 建立了什么关系

### 2.1 连接与发射是两个动作

```cpp
QObject::connect(sender, &Sender::valueChanged,
                 receiver, &Receiver::setValue,
                 Qt::AutoConnection);
```

| 部分 | 含义 |
| --- | --- |
| `sender` | 发出信号的对象指针 |
| `&Sender::valueChanged` | 信号的成员函数指针；这里不加调用括号 |
| `receiver` | 接收对象，影响连接生命周期和队列投递线程 |
| `&Receiver::setValue` | 处理信号的成员函数指针 |
| `Qt::AutoConnection` | 调用策略，省略时默认使用它 |

`connect()` 建立关系并返回 `QMetaObject::Connection`，不会立即执行槽。之后执行：

```cpp
emit valueChanged(42);
```

才会触发相应连接。`emit` 是 Qt 的语法标记，不是创建线程或启动异步任务的关键字；是否排队由连接方式决定。

在 `QObject` 派生类成员函数里通常直接写 `connect(...)`；在 `main()` 等位置写 `QObject::connect(...)` 更清楚。

### 2.2 一对多、多对一与返回值

一个信号可以连接多个槽，一个槽也能接收多个信号。没有接收者时，发出信号也合法。

信号通常声明为 `void`。不要把信号槽当成收集槽返回值的接口，异步结果应使用结果信号、回调或其他明确的异步协议返回。

```cpp
auto connection = QObject::connect(sender, &Sender::valueChanged,
                                   receiver, &Receiver::setValue);
if (!connection)
    qWarning() << "连接建立失败";
```

对于现代模板写法，很多类型错误会在编译期报错；字符串查找失败、重复的唯一连接等则可能得到无效连接句柄。

## 3. 现代 connect 的各种写法

本节假定存在下列两个类，`sender`、`receiver` 分别是它们的有效对象指针：

```cpp
#include <QObject>
#include <QDebug>
#include <QString>

class Sender : public QObject
{
    Q_OBJECT
public:
    using QObject::QObject;
signals:
    void valueChanged(int value);
    void messageReady(const QString &text);
    void finished();
};

class Receiver : public QObject
{
    Q_OBJECT
public:
    using QObject::QObject;
    void save(int value) { qDebug() << "保存" << value; }
public slots:
    void setValue(int value) { qDebug() << "更新" << value; }
    void refresh() { qDebug() << "刷新"; }
};
```

### 3.1 连接到成员槽或普通成员函数

```cpp
QObject::connect(sender, &Sender::valueChanged,
                 receiver, &Receiver::setValue);

QObject::connect(sender, &Sender::valueChanged,
                 receiver, &Receiver::save);
```

`save()` 没有放在 `slots:` 下，也能通过函数指针连接。现代写法直接掌握 C++ 函数类型，不要求接收方法一定注册成元对象槽。

| 声明或机制 | 与 connect 的关系 |
| --- | --- |
| `Q_OBJECT` | 自定义信号类需要它，并需要 moc 生成代码；接收普通成员函数不要求类为了该方法额外声明 `slots` |
| `signals:` | 声明信号，由 moc 提供实现，通常不要自己实现信号体 |
| `slots:` | 把方法注册到元对象系统，支持字符串查找等；方法实现仍需自己编写 |
| 普通成员函数 | 可作为现代函数指针连接的接收方法，但不能直接用 `SLOT(save(int))` 查找到它 |
| `Q_INVOKABLE` | 把方法暴露给元对象调用等机制，不是现代 connect 的必需条件，也不会让普通方法变成信号 |

接收对象仍需是 `QObject` 派生对象。只有普通 C++ 对象时，可通过带 context 的 Lambda 转发，并自行保证普通对象的生命周期。

### 3.2 连接到带 context 的 Lambda

```cpp
QObject::connect(sender, &Sender::valueChanged,
                 receiver, [receiver](int value) {
    receiver->save(value * 2);
});
```

这里 `receiver` 是 **context（上下文对象）**，Lambda 才是实际回调。context 的作用有两项：

1. sender 或 context 销毁时，连接自动断开。
2. Auto / Queued 等策略需要投递时，投递到 context 所属线程。

context 不会因为捕获列表中出现某个指针就自动改变；捕获了 `receiver` 和把它传为 context 是两个不同动作。

```cpp
// 位于 QObject 派生类的成员函数中：this 同时出现在 context 与捕获列表。
connect(sender, &Sender::valueChanged, this, [this](int value) {
    updateView(value); // 假设当前类有该方法
}, Qt::QueuedConnection);
```

### 3.3 不带 context 的 Lambda

```cpp
QObject::connect(sender, &Sender::valueChanged, [](int value) {
    qDebug() << value;
});
```

这个三参数重载始终使用直接连接：回调在发射信号的线程执行。它没有 context，也没有可直接传入的连接类型参数；需要选择类型时，改用上一节的带 context 重载。

sender 销毁会断开连接，但 Lambda 捕获的其他对象销毁不会自动断开。例如：

```cpp
// 风险示例：this 仅被捕获，未成为 context。
connect(sender, &Sender::finished, [this] {
    refresh();
});
```

如果 sender 比 `this` 活得更久，就可能访问已销毁对象；如果信号从后台线程发出，还可能错误地在后台刷新界面。通常应写成 `connect(sender, &Sender::finished, this, [this] {...})`。

Qt 6.7 起可在工程中定义 `QT_NO_CONTEXTLESS_CONNECT`，禁用该三参数重载，帮助约束代码风格。参考：[QObject 的 contextless connect 说明](https://doc.qt.io/qt-6.8/qobject.html#QT_NO_CONTEXTLESS_CONNECT)。

### 3.4 自由函数与函数对象

以下代码中的定义放在文件作用域，连接语句放在函数内：

```cpp
void printValue(int value)
{
    qDebug() << "自由函数" << value;
}

struct ValuePrinter
{
    void operator()(int value) const
    {
        qDebug() << "函数对象" << value;
    }
};
```

```cpp
QObject::connect(sender, &Sender::valueChanged,
                 receiver, &printValue, Qt::QueuedConnection);

QObject::connect(sender, &Sender::valueChanged,
                 receiver, ValuePrinter{}, Qt::AutoConnection);
```

函数没有所属线程，但这两条连接有 context，因此排队时使用 `receiver` 的线程。静态成员函数也可按自由函数理解。context 不会自动管理函数对象内部持有的其他裸指针。

### 3.5 信号连接到信号

假设 `relay` 也是 `Sender` 对象：

```cpp
QObject::connect(sender, &Sender::valueChanged,
                 relay, &Sender::valueChanged);
```

sender 的信号触发后，relay 会继续发出自己的信号，适合组件转发事件。两段连接各自决定调用策略。避免无条件构建“甲转发给乙，乙又转发给甲”的循环，否则可能递归或不断排队。

## 4. 参数匹配、重载与旧式连接

### 4.1 槽可以少接参数，不能任意挑选参数

```cpp
QObject::connect(sender, &Sender::valueChanged,
                 receiver, &Receiver::refresh); // 丢弃 int 参数
```

如果信号是 `(int, QString)`，直接连接的槽可以接 `()`、`(int)`、兼容的 `(int, QString)`，不能只写 `(QString)` 来选第二个参数。此时用 Lambda 适配：

```cpp
// 假设 source 的信号为 dataReady(int id, const QString &name)。
QObject::connect(source, &Source::dataReady,
                 receiver, [receiver](int, const QString &name) {
    qDebug() << receiver << name;
});
```

现代连接允许兼容的隐式类型转换，例如 `int` 到 `double`；字符串方式依赖元对象签名匹配，不具备同样的 C++ 转换能力。为便于阅读，涉及业务意义的转换仍建议显式写在 Lambda 中。

### 4.2 重载函数必须消除歧义

假设 `Device` 有 `changed(int)` 和 `changed(const QString &)` 两个信号：

```cpp
QObject::connect(device, qOverload<int>(&Device::changed),
                 receiver, &Receiver::setValue);
```

等价的标准 C++ 写法：

```cpp
QObject::connect(device,
    static_cast<void (Device::*)(int)>(&Device::changed),
    receiver, &Receiver::setValue);
```

接收方法重载时也需要消歧；const 与非 const 重载可用 `qConstOverload` / `qNonConstOverload` 或准确的成员函数指针类型。`qOverload` 可用于 Qt 5.7 及之后版本。

### 4.3 默认参数不缩短函数指针签名

假设接收方法是：

```cpp
void reset(int mode = 0);
```

它的成员函数指针仍包含一个 `int` 参数，不能把无参 `finished()` 直接连接到它并指望模板补默认值。用 Lambda 发起普通 C++ 调用即可：

```cpp
// 假设 receiver 所属类声明了 reset(int mode = 0)。
QObject::connect(sender, &Sender::finished, receiver, [receiver] {
    receiver->reset();
});
```

### 4.4 SIGNAL() / SLOT() 字符串连接

```cpp
QObject::connect(sender, SIGNAL(valueChanged(int)),
                 receiver, SLOT(setValue(int)),
                 Qt::QueuedConnection);
```

宏参数写方法签名，不写参数名，不写函数调用实参。该方式在运行时查找元对象，拼写错误可能只产生运行时警告。

| 对比项 | 函数指针 / Functor 写法 | 字符串写法 |
| --- | --- | --- |
| 签名检查 | 主要在编译期 | 主要在运行期 |
| Lambda / 自由函数 | 支持 | 不能直接连接 |
| 普通成员方法 | 支持 | `SLOT()` 要求注册为槽 |
| 隐式类型转换 | 支持兼容转换 | 依赖签名匹配 |
| 默认参数 | 仍按完整函数指针检查，可用 Lambda 适配 | 已注册的带默认参数槽可提供省略尾部参数的元对象签名 |
| QML 动态定义的方法 | 不能凭不存在的 C++ 成员指针连接 | 某些与 QML 交互场景可使用运行时签名 |

函数指针方式从 Qt 5 起已支持。维护旧代码时需要读懂字符串方式，新写 C++ 代码通常优先函数指针与带 context 的 Lambda。

参考：[Qt 两种连接语法的区别](https://doc.qt.io/qt-6.8/signalsandslots-syntaxes.html)。

## 5. 六个枚举项：基础策略与附加标志

`Qt::ConnectionType` 的四个基础策略决定执行时机，另外两个枚举项是附加标志。

| 类型 | 数值 | 槽在哪执行 | 执行时机 | 发射线程是否等待槽完成 |
| --- | --- | --- | --- | --- |
| `AutoConnection` | `0` | 按本次发射线程与接收线程的关系决定 | 同线程直接，跨线程排队 | 取决于实际选择 |
| `DirectConnection` | `1` | 发射信号的当前线程 | 发射过程中立即调用 | 是，同步调用 |
| `QueuedConnection` | `2` | receiver / context 所属线程 | 由该线程的事件循环调度 | 否，不等待该槽完成 |
| `BlockingQueuedConnection` | `3` | receiver / context 所属线程 | 投递后等待接收线程执行 | 是，阻塞等待 |
| `UniqueConnection` | `0x80` | 由基础策略决定 | 阻止相同成员函数连接重复建立 | 由基础策略决定 |
| `SingleShotConnection` | `0x100` | 由基础策略决定 | 首次触发后自动断开 | 由基础策略决定 |

“不等待”不代表没有投递、复制和调度开销；同一个信号上的其他直接连接仍可能让整次 `emit` 耗时。

官方定义：[Qt::ConnectionType](https://doc.qt.io/qt-6.8/qt.html#ConnectionType-enum)。以下章节用具体时序解释其用法。

## 6. AutoConnection：默认方式怎样判断线程

```cpp
QObject::connect(sender, &Sender::valueChanged,
                 receiver, &Receiver::setValue);
```

判断发生在每次信号发出时，可以概括为：

```text
发射时的 QThread::currentThread() == receiver->thread()？
    是 → 直接调用
    否 → 排队到 receiver 所属线程
```

注意这里比较的是**实际发射信号的线程**与接收对象所属线程，不是简单比较 `sender->thread()` 和 `receiver->thread()`，也不是看当初在哪个线程执行 `connect()`。

例如，一个 `QThread` 对象本身在主线程创建，但它的 `started()`、`finished()` 信号在所管理的线程中发出。判断 Auto 时要看信号实际从哪里发出。

这项规则也不意味着可以随意从任何线程调用其他对象的方法；对象内部状态是否可被跨线程访问，仍需单独判断。

常规应用内部通知通常先用 Auto。若业务明确要求“即使同线程也延后处理”，应显式选择 Queued。

## 7. DirectConnection：立即执行与重入

```cpp
QObject::connect(sender, &Sender::valueChanged,
                 receiver, &Receiver::setValue,
                 Qt::DirectConnection);
```

调用关系近似为：

```text
发射线程：emit valueChanged(42)
              → receiver->setValue(42)
              ← 槽返回
          继续 emit 之后的语句
```

它不会切换到 receiver 所属线程。如果后台线程发信号，强制 Direct 后，槽就在后台线程运行；receiver 在主线程也不会改变这一点。

因此，不能为了“更快更新”而让工作线程直接操作 GUI。大多数 QObject 实例也不支持多个线程同时调用其有状态方法。`connect()` 本身线程安全，不代表槽里的业务操作线程安全。

直接连接还要注意重入：槽可以再次触发信号，甚至调用回发送者。发射信号前应先完成必须成立的状态更新，避免槽看到半更新的数据。持锁发信号时，如果槽再次申请同一把非递归锁，也可能死锁。

Direct 适合同线程、需要同步完成且调用关系清楚的通知；普通场景通常让 Auto 决定即可。

## 8. QueuedConnection：把一次调用投递给事件循环

### 8.1 排队的是什么

```cpp
QObject::connect(sender, &Sender::valueChanged,
                 receiver, &Receiver::setValue,
                 Qt::QueuedConnection);
```

可以按下图理解：

```text
发射线程                        receiver 所属线程
emit valueChanged(42)
    → 保存本次调用参数
    → 投递调用事件 ─────────────→ 事件队列
继续执行后续代码                    ↓ 事件循环取出事件
                                 setValue(42)
```

这是队列调用，不是为这次信号创建新线程。receiver 属于主线程时，槽仍在主线程执行，重计算仍会卡界面。

### 8.2 同线程也能排队

同线程显式指定 Queued，可以把工作延后到事件处理阶段，避免当前调用栈中直接进入接收者。

例如，在模型完成一批状态更新之后再响应通知，或者避免信号立即回调造成的递归。它不会自动合并同类事件：连续发出 100 次信号，通常会产生 100 次调用。

如果接收线程就是发射线程，而且当前路径没有嵌套事件循环或 `processEvents()`，槽会在当前发射过程及紧随其后的同步代码完成后才有机会执行。若主动启动嵌套事件循环，队列调用可能在外层函数尚未返回时发生，因此不能把“队列”理解成绝对不会重入。

跨线程时，接收线程可能立即处理事件，甚至在发射线程执行到下一条语句前就完成槽调用。排队保证的是线程投递方式，不是两个线程日志之间的全局先后顺序。

### 8.3 接收线程必须能处理事件

GUI 主线程通常通过 `app.exec()` 处理事件。默认 `QThread::run()` 也会进入事件循环，但自定义 `run()` 时，如果只计算、未调用 `exec()`，不能指望该线程处理普通队列槽。

事件循环被长任务占住时，排队调用要等待；线程已停止时，不会正常消费新投递的调用。仅仅拥有一个 `QThread` 对象并不等于后台事件循环已经运行。

### 8.4 积压、顺序与吞吐量

如果每秒发出 1000 次，而每次槽处理需要 10 毫秒，接收线程会越来越落后。Queued 不会自动丢弃旧结果，也没有内置的业务背压策略。

可按业务选择降低发射频率、批量传递、只保留最新状态，或用请求编号忽略过期结果。需要处理每一条交易与仅展示最新进度的场景，应采用不同策略。

同一信号连接多个槽时，Qt 按连接建立顺序激活连接。对于同一线程依次投递给同一接收线程的普通队列调用，可按投递顺序理解；不同发射线程、不同接收线程及其他事件交错时，不能据此推导所有槽的全局完成顺序。不要把混用 Direct 与 Queued 的槽组织成隐含业务流水线。

线程和事件循环参考：[Threads and QObjects](https://doc.qt.io/qt-6.8/threads-qobject.html)。

## 9. BlockingQueuedConnection：跨线程同步等待

假设 `controller` 的 `flushRequested()` 连接到后台 `worker` 的 `flush()`：

```cpp
QObject::connect(controller, &Controller::flushRequested,
                 worker, &Worker::flush,
                 Qt::BlockingQueuedConnection);
```

```text
发射线程                         worker 线程
emit flushRequested()
    → 投递调用 ─────────────────→ flush()
    → 阻塞等待                      ↓ 完成
    ←──────────────────────────── 通知等待方
继续执行后续代码
```

使用前必须明确：

- 发射线程与接收对象所属线程不同，否则会死锁。
- 接收线程事件循环正在运行，而且能处理这次调用。
- 槽不会等待发射线程处理事件，也不会等待发射线程持有的锁。
- 阻塞时长在业务上可以接受；该连接策略没有可配置的等待超时。

典型死锁包括“主线程等待 worker，worker 又请求主线程并等待”，以及“发射者持有锁，接收槽获取同一把锁”。主线程使用 BlockingQueued 时，即使最终能返回，等待期间界面也会失去响应。

它适合少量、受控的跨线程同步操作。日常后台任务优先使用“排队请求 + 结果信号”。BlockingQueued 不会把槽返回值变成信号的返回值，也不提供任务取消机制。

同线程使用 BlockingQueued 的代码只作为错误概念说明，不应运行验证。

## 10. UniqueConnection 与 SingleShotConnection

### 10.1 UniqueConnection：拒绝重复连接

默认情况下，相同连接建立两次，信号触发时槽就会执行两次。Unique 用于防止同一发送对象、同一信号、同一接收对象、同一成员接收函数的重复连接。

```cpp
auto first = QObject::connect(sender, &Sender::valueChanged,
                              receiver, &Receiver::setValue,
                              Qt::UniqueConnection);
auto second = QObject::connect(sender, &Sender::valueChanged,
                               receiver, &Receiver::setValue,
                               Qt::UniqueConnection);
// 假设此前没有相同连接：first 有效，second 无效。
```

仅传 Unique 时，基础策略为 Auto。它不负责“相同值只通知一次”，不去重信号参数，也不能用于把已有连接从 Auto 更新成 Queued。要更改策略，先断开旧连接再建立新连接。

**Unique 不支持 Lambda、自由函数和函数对象，只用于成员函数接收端。** 即使两次 Lambda 文本完全相同，也不能依靠 Unique 去重。Lambda 应通过“只初始化一次”或保存句柄、重连前断开的方式管理。

### 10.2 SingleShotConnection：仅保留一次触发

```cpp
QObject::connect(sender, &Sender::finished, receiver, [] {
    qDebug() << "只响应一次完成事件";
}, Qt::SingleShotConnection);
```

这是 Qt 6.0 起提供的标志，支持 Lambda。它在首次触发连接时自动断开，避免后续信号再次通过该连接触发槽。

与 Queued 组合时，首次触发安排一次调用，后续发射不会继续给这个连接追加调用；如果 context 在执行前销毁，回调仍可能一次也不执行。“单次”不是一定执行成功的承诺。

| 机制 | “单次”的对象 |
| --- | --- |
| `Qt::SingleShotConnection` | 这条信号槽连接只触发一次 |
| `QTimer::singleShot()` | 安排一次延迟回调 |
| `timer->setSingleShot(true)` | 这次启动的定时器到期后停止，可再次启动 |

SingleShotConnection 不会销毁 sender / receiver；连接到一个周期定时器上也不会停止定时器本身。

### 10.3 正确组合标志

位或之后显式转回 `Qt::ConnectionType`，清楚表达“一种基础策略 + 附加标志”：

```cpp
const auto queuedUnique = static_cast<Qt::ConnectionType>(
    Qt::QueuedConnection | Qt::UniqueConnection);

QObject::connect(sender, &Sender::valueChanged,
                 receiver, &Receiver::setValue, queuedUnique);

const auto queuedOnce = static_cast<Qt::ConnectionType>(
    Qt::QueuedConnection | Qt::SingleShotConnection);

QObject::connect(sender, &Sender::finished,
                 receiver, [] { qDebug() << "单次排队响应"; }, queuedOnce);
```

成员函数接收端也可同时叠加 Unique 和 SingleShot。不要组合两种基础策略：例如 `DirectConnection | QueuedConnection` 数值恰好为 `3`，会变成 BlockingQueuedConnection，不是“先直接再排队”。

Qt 5 没有 SingleShotConnection。维护 Qt 5 工程时需要显式管理单次状态和连接句柄；对于队列连接，已经入队的多次调用还需用状态保护，不能仅在第一次回调里 disconnect 就假定其余调用必定取消。

去重限制参考：[QObject::connect](https://doc.qt.io/qt-6.8/qobject.html#connect-2)，单次标志定义见 [Qt::ConnectionType](https://doc.qt.io/qt-6.8/qt.html#ConnectionType-enum)。

## 11. 参数、线程归属与生命周期

### 11.1 队列参数为什么需要元类型

调用稍后发生时，发射函数的局部变量可能已经离开作用域。因此普通 Queued 调用需要保存一份可用于后续执行的参数。推荐传递明确、可复制的值类型：

```cpp
struct Result
{
    int requestId = 0;
    QString text;
};
Q_DECLARE_METATYPE(Result) // 放在类型定义后；命名空间类型应在命名空间外使用全名
```

在建立连接之前注册：

```cpp
qRegisterMetaType<Result>("Result");
```

`Q_DECLARE_METATYPE` 让模板元类型设施认识类型；`qRegisterMetaType` 注册运行时类型信息，尤其有利于队列和字符串签名场景。Qt 6 的部分类型化路径能自动取得类型信息，不意味着任意自定义类型、任意连接路径都不需要注册。教学和跨版本代码显式声明、注册更清楚。

`int`、`QString` 等内置类型通常无需手动注册。自定义类型应完整定义并满足相关复制要求；不要尝试通过普通 Queued 传递不可复制的 `std::unique_ptr`，或把非常量引用当成跨线程输出参数。

信号写成 `void ready(const Result &result)` 也不意味着普通 Queued 会保留对原栈变量的引用；Qt 会为队列事件保存参数。反过来，若参数本身是裸指针，复制的是地址：

```cpp
// 假设信号为 ready(MyData *data)。
// 指针入队不延长 *data 生命周期，也不阻止其他线程同时修改它。
```

大对象可以考虑不可变数据或合适的共享所有权类型，但“共享所有权”不等于“共享对象内部可并发修改”。QString 等隐式共享类型的值复制也不必然立即深复制所有底层数据。

参考：[Q_DECLARE_METATYPE](https://doc.qt.io/qt-6.8/qmetatype.html#Q_DECLARE_METATYPE)、[qRegisterMetaType](https://doc.qt.io/qt-6.8/qmetatype.html#qRegisterMetaType)。

### 11.2 QThread 对象和工作线程不是一回事

```cpp
QThread thread;            // 假设此语句在主线程运行：这个 QObject 属于主线程
auto *worker = new Worker; // 初始也属于主线程，不设置 parent
worker->moveToThread(&thread);
thread.start();            // 启动由 thread 管理的后台执行线程
```

移动后 `worker->thread()` 为 `&thread`，但 `thread.thread()` 仍指向创建它的主线程。把槽写在 `QThread` 子类里，再给该对象发送 Auto 通知，不能因此假定槽会在 `run()` 所在线程执行。

还需注意：

- 有 parent 的 QObject 不能直接移动到其他线程；子对象会随父对象移动。
- 对象移动通常应由其当前所属线程发起；不能任意从另一线程把对象“拉过来”。
- 普通 C++ 指针成员不等于 QObject 子对象。未建立父子关系的 QObject 成员不会仅因外层对象移动而自动移动。
- `moveToThread()` 不会把普通方法调用自动变成队列调用；主线程直接执行 `worker->doWork()`，仍然在主线程运行。
- Widgets 界面对象应留在主线程；后台计算通过信号传回数据，由主线程更新界面。

Qt 6.8 的 `moveToThread()` 返回 bool，可检查是否成功；Qt 6.7 之前返回 void。参考：[QObject::moveToThread](https://doc.qt.io/qt-6.8/qobject.html#moveToThread)、[QThread 详解](https://doc.qt.io/qt-6.8/qthread.html#details)。

### 11.3 Lambda 捕获不等于生命周期管理

```cpp
// 局部示例：context 是一个寿命足够长的 QObject。
QString name = "张三";
QObject::connect(sender, &Sender::finished, context, [name] {
    qDebug() << name; // 保存 QString 值，原局部变量离开作用域仍可使用
});
```

如果改成 `[&name]`，回调稍后执行时，原局部变量可能已经销毁。`[this]` 保存的也只是指针，不会延长当前对象寿命；其他裸指针按值捕获同理。

context 只对它自身的连接生命周期和投递线程负责。如果 Lambda 还访问另一个对象，应让所有权关系保证该对象存活，或用 `QPointer` 检查 QObject 是否仍存在。`QPointer` 不会赋予跨线程并发访问权限，也不能替代线程同步。

### 11.4 保存句柄与断开连接

```cpp
QMetaObject::Connection connection = QObject::connect(
    sender, &Sender::valueChanged,
    receiver, &Receiver::setValue);

QObject::disconnect(connection); // 精确断开这一条连接
```

也可以按信号与成员函数匹配：

```cpp
QObject::disconnect(sender, &Sender::valueChanged,
                    receiver, &Receiver::setValue);
```

后一种会断开匹配的连接，包括重复建立的多条连接。Lambda 通常直接保存句柄最方便，不要重新构造一个“看起来一样”的 Lambda 来尝试匹配。

`QMetaObject::Connection` 是句柄，不是自动解绑的作用域守卫：句柄变量销毁、被覆盖或丢弃，不会自动断开底层连接。若需要自动解绑，应实现明确的 RAII 管理。

句柄转 bool 适合检查建立连接是否成功，不应把它当成持续监测 sender / receiver 是否仍存活的工具。参考：[QMetaObject::Connection](https://doc.qt.io/qt-6.8/qmetaobject-connection.html)。

### 11.5 自动断开与已排队调用的边界

sender 或 receiver / context 销毁，会自动断开相关连接；接收对象销毁时，Qt 还会移除投递给它的待处理事件。接收者的销毁必须遵守其线程和生命周期规则，跨线程清理通常使用 `deleteLater()`。

但显式调用 `disconnect()` 不等于撤回全部已排队调用。已经安排的队列事件仍可能被执行；正在执行的槽也不会被中断。sender 销毁也不应被当成撤回接收端已有业务结果的机制。

对于“刷新后旧结果不能覆盖新结果”，在结果中携带请求编号，并在接收时检查当前编号；对于取消任务，设计取消标记或中断检查。连接控制与业务取消是不同层面的职责。

参考：[QObject::disconnect 的队列调用说明](https://doc.qt.io/qt-6.8/qobject.html#disconnect)。

## 12. 三个可以独立运行的完整示例

### 12.1 通用构建配置

每次只选择下面一个完整 `main.cpp`，与这份 `CMakeLists.txt` 放入同一目录：

```cmake
cmake_minimum_required(VERSION 3.16)
project(ConnectDemo LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_AUTOMOC ON)

find_package(Qt6 6.8 REQUIRED COMPONENTS Core)
add_executable(connect_demo main.cpp)
target_link_libraries(connect_demo PRIVATE Qt6::Core)
```

构建并运行，Qt 路径按本机实际位置调整：

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug -DCMAKE_PREFIX_PATH="$HOME/Qt/6.8.3/macos"
cmake --build build
./build/connect_demo
```

Windows 使用对应安装路径，并按生成器的配置目录查找可执行文件；麒麟 / Linux 同样将前缀换成实际 Qt 安装目录。本文不需要额外安装 Qt Test。

类定义在 `main.cpp` 中且包含 `Q_OBJECT`，因此文件结尾使用 `#include "main.moc"`，由 CMake AUTOMOC 生成实现；不要自己创建这个 moc 文件。

### 12.2 示例一：同线程 Auto / Direct / Queued 的日志顺序

完整 `main.cpp`：

```cpp
#include <QCoreApplication>
#include <QDebug>
#include <QObject>
#include <QStringList>
#include <QThread>

class Emitter : public QObject
{
    Q_OBJECT
signals:
    void fired(int value);
};

int main(int argc, char *argv[])
{
    QCoreApplication app(argc, argv);
    Emitter emitter;
    QStringList order;

    QObject::connect(&emitter, &Emitter::fired, &app, [&](int value) {
        Q_ASSERT(QThread::currentThread() == app.thread());
        order << "auto";
        qDebug() << "Auto" << value;
    });
    QObject::connect(&emitter, &Emitter::fired, &app, [&](int value) {
        order << "direct";
        qDebug() << "Direct" << value;
    }, Qt::DirectConnection);
    QObject::connect(&emitter, &Emitter::fired, &app, [&](int value) {
        Q_ASSERT(QThread::currentThread() == app.thread());
        order << "queued";
        qDebug() << "Queued" << value;
        Q_ASSERT(order == QStringList({"before", "auto", "direct",
                                      "after", "queued"}));
        app.quit();
    }, Qt::QueuedConnection);

    order << "before";
    qDebug() << "emit 前";
    emit emitter.fired(42);
    order << "after";
    qDebug() << "emit 后";
    return app.exec();
}

#include "main.moc"
```

预期输出顺序：

```text
emit 前
Auto 42
Direct 42
emit 后
Queued 42
```

三个槽都在主线程运行。Queued 的区别在执行时机，不能从“延后执行”推断“去了后台”。这里在 `app.exec()` 前已经投递了调用，进入事件循环后才处理它。

### 12.3 示例二：Worker + QThread + 自定义结果类型

主线程发出请求，后台生成结果，主线程接收结果后要求线程退出。完整 `main.cpp`：

```cpp
#include <QCoreApplication>
#include <QDebug>
#include <QMetaType>
#include <QObject>
#include <QString>
#include <QThread>
#include <QTimer>

struct Result
{
    int requestId = 0;
    QString text;
};
Q_DECLARE_METATYPE(Result)

class Controller : public QObject
{
    Q_OBJECT
signals:
    void workRequested(int requestId);
};

class Worker : public QObject
{
    Q_OBJECT
public slots:
    void doWork(int requestId)
    {
        Q_ASSERT(QThread::currentThread() == thread());
        qDebug() << "后台执行" << QThread::currentThread()->objectName();
        const Result result{requestId, QStringLiteral("计算完成")};
        emit resultReady(result);
    }
signals:
    void resultReady(const Result &result);
};

int main(int argc, char *argv[])
{
    QCoreApplication app(argc, argv);
    QThread::currentThread()->setObjectName("主线程");
    qRegisterMetaType<Result>("Result");

    Controller controller;
    QThread workerThread;
    workerThread.setObjectName("工作线程");
    auto *worker = new Worker; // 不设置主线程 parent，才能移动
    if (!worker->moveToThread(&workerThread)) {
        delete worker;
        return 1;
    }

    bool received = false;
    QObject::connect(&controller, &Controller::workRequested,
                     worker, &Worker::doWork, Qt::QueuedConnection);
    QObject::connect(worker, &Worker::resultReady, &app,
                     [&](const Result &result) {
        Q_ASSERT(QThread::currentThread() == app.thread());
        Q_ASSERT(result.requestId == 7);
        Q_ASSERT(result.text == QStringLiteral("计算完成"));
        received = true;
        qDebug() << "主线程收到" << result.requestId << result.text
                 << QThread::currentThread()->objectName();
        workerThread.quit();
    }, Qt::QueuedConnection);

    // 使用 Qt 官方推荐的结束清理模式，不强制改成 QueuedConnection。
    QObject::connect(&workerThread, &QThread::finished,
                     worker, &QObject::deleteLater);
    QObject::connect(&workerThread, &QThread::finished,
                     &app, &QCoreApplication::quit, Qt::QueuedConnection);

    // 演示程序的失败兜底；quit 不会强行中止正在运行的耗时槽。
    QTimer::singleShot(5000, &app, [&] {
        qWarning() << "等待结果超时";
        workerThread.quit();
        app.exit(1);
    });

    workerThread.start();
    emit controller.workRequested(7);
    const int code = app.exec();
    workerThread.quit();
    workerThread.wait(); // QThread 栈对象销毁前，确保后台线程已经结束
    return received ? code : 1;
}

#include "main.moc"
```

典型输出：

```text
后台执行 "工作线程"
主线程收到 7 "计算完成" "主线程"
```

数据流为：

```text
Controller（主线程）
    workRequested(7)
        ↓ 队列投递
Worker（工作线程）
    doWork(7) → resultReady(Result)
        ↓ 队列投递，保存结果值
app context（主线程）
    处理结果 → workerThread.quit()
        ↓
工作线程 finished → worker.deleteLater()
        ↓
主线程退出事件循环 → wait() → 安全销毁 QThread
```

`finished()` 发出时，后台普通事件循环已经停止，但 Qt 仍会处理延迟删除事件，因此这条默认 Auto 的 `finished → deleteLater` 连接是官方支持的清理模式。不要为了“跨线程统一写法”强行将它改成普通 Queued，以免 `deleteLater()` 的调用本身还要等待已停止的循环。

这里的 `doWork()` 会及时返回。若真实任务长时间不返回，`quit()` 不会抢占中止它，兜底定时器也不能保证随后 `wait()` 有限时完成。长任务需定期检查中断请求，或采用可取消的分段工作；不要用 `terminate()` 代替正常清理。

参考：[QThread::finished 与延迟删除](https://doc.qt.io/qt-6.8/qthread.html#finished)。

### 12.4 示例三：唯一连接与单次队列连接

完整 `main.cpp`：

```cpp
#include <QCoreApplication>
#include <QDebug>
#include <QEvent>
#include <QObject>

class Emitter : public QObject
{
    Q_OBJECT
signals:
    void fired(int value);
};

class Counter : public QObject
{
    Q_OBJECT
public:
    int count = 0;
    void record(int value)
    {
        ++count;
        qDebug() << "成员函数" << value;
    }
};

int main(int argc, char *argv[])
{
    QCoreApplication app(argc, argv);
    Emitter emitter;
    Counter counter;
    int onceCount = 0;
    int firstValue = 0;

    const auto uniqueType = static_cast<Qt::ConnectionType>(
        Qt::DirectConnection | Qt::UniqueConnection);
    const auto first = QObject::connect(&emitter, &Emitter::fired,
        &counter, &Counter::record, uniqueType);
    const auto duplicate = QObject::connect(&emitter, &Emitter::fired,
        &counter, &Counter::record, uniqueType);
    if (!first || duplicate)
        return 1;

    const auto onceType = static_cast<Qt::ConnectionType>(
        Qt::QueuedConnection | Qt::SingleShotConnection);
    QObject::connect(&emitter, &Emitter::fired, &app, [&](int value) {
        ++onceCount;
        firstValue = value;
        qDebug() << "单次队列回调" << value;
    }, onceType);

    emit emitter.fired(10);
    emit emitter.fired(20);
    if (counter.count != 2 || onceCount != 0)
        return 1;

    // 仅用于确定性演示：在当前线程显式派发 app 的 MetaCall 事件。
    // 正常应用一般直接使用 app.exec()，无需手工派发。
    QCoreApplication::sendPostedEvents(&app, QEvent::MetaCall);
    if (onceCount != 1 || firstValue != 10)
        return 1;
    qDebug() << "验证通过：成员函数 2 次，单次回调 1 次";
    return 0;
}

#include "main.moc"
```

连续发射两次时，成员函数响应两次；第二次重复建立的连接被拒绝，所以不会响应四次。单次队列连接仅处理第一次的参数 `10`，没有为第二次的 `20` 再安排回调。

## 13. 运行时发现连接与自动连接

### 13.1 QMetaMethod：运行时查找信号和槽

需要按运行时签名连接时，可以取得两个 `QMetaMethod`。以下复用第 3 节的 `sender`、`receiver`：

```cpp
#include <QMetaMethod>

const QMetaObject *sourceMeta = sender->metaObject();
const QMetaObject *targetMeta = receiver->metaObject();
const int signalIndex = sourceMeta->indexOfSignal("valueChanged(int)");
const int slotIndex = targetMeta->indexOfSlot("setValue(int)");
if (signalIndex >= 0 && slotIndex >= 0) {
    const auto connection = QObject::connect(
        sender, sourceMeta->method(signalIndex),
        receiver, targetMeta->method(slotIndex), Qt::AutoConnection);
    if (!connection)
        qWarning() << "运行时连接失败";
}
```

适合动态工具和反射场景。它仍受签名兼容、元对象注册和连接类型约束；普通 `save()` 不会因可用函数指针连接，就自动出现在 `indexOfSlot()` 中。普通业务代码无须绕到这一层。

参考：[QObject 的 QMetaMethod 连接重载](https://doc.qt.io/qt-6.8/qobject.html#connect)。

### 13.2 connectSlotsByName：按对象名约定连接

Qt Widgets Designer 生成的 `setupUi()` 通常会调用：

```cpp
QMetaObject::connectSlotsByName(this);
```

它会根据子对象的 `objectName` 查找形如 `on_<对象名>_<信号名>(...)` 的槽。例如按钮名为 `saveButton`，父界面的槽可以声明为：

```cpp
private slots:
    void on_saveButton_clicked();
```

这依赖对象名和元对象槽签名，不同于 `Qt::AutoConnection`：前者是“自动找到连接目标”，后者是“已有连接如何执行”。如果已经自动连接，又手工 connect 同一逻辑，可能造成一次点击执行两次。

参考：[QMetaObject::connectSlotsByName](https://doc.qt.io/qt-6.8/qmetaobject.html#connectSlotsByName)。

### 13.3 与 QML Connections 的边界

QML 中的 `onClicked`、`Connections { target: ... }` 用于声明信号处理器，不直接采用 C++ 的五参数语法。本文关于 `Qt::ConnectionType` 的选择针对 C++ `QObject::connect()`；不要直接向 QML `Connections` 添加 `QueuedConnection` 属性。

QML 分层事件流见：[Qt QML Signal 与分层事件流](Qt%20QML%20Signal与分层事件流.md)。

## 14. 常见问题排查

| 现象 | 优先检查 | 处理方向 |
| --- | --- | --- |
| 槽始终不执行 | 是否发出信号、连接是否成功、是否在连接前已发出信号 | 先加发射日志并检查句柄；连接不重放历史信号 |
| Queued 槽不执行 | 接收线程是否启动事件循环、是否被长任务占住 | 检查 `receiver->thread()` 和线程执行路径 |
| 一次点击执行多次 | 重复初始化、自动连接与手工连接并存 | 初始化只做一次；成员函数可用 Unique；Lambda 管理句柄 |
| `Cannot queue arguments of type ...` | 自定义类型是否完整定义、已声明和注册 | 在连接前 `qRegisterMetaType`，核对运行时类型名 |
| 编译提示无法匹配 connect | 信号 / 槽重载、参数数量或类型不匹配 | 使用 `qOverload` 或 Lambda 适配 |
| `No such slot` | 字符串签名拼写、是否注册为槽、moc 是否更新 | 核对签名，优先改用现代写法 |
| 修改 Q_OBJECT 后链接错误 | 是否开启 AUTOMOC、是否缺少生成代码 | 检查构建配置和类所在源文件 |
| 已经 Queued 仍卡界面 | context 是否属于主线程、主线程槽是否耗时 | 将计算放到 Worker，主线程只处理必要界面更新 |
| 槽在线程错误的位置运行 | 强制 Direct、无 context Lambda、直接调用 worker 方法 | 明确 context 和投递策略，记录实际执行线程 |
| BlockingQueued 卡住 | 同线程调用、循环等待、锁冲突、接收循环停止 | 改成异步请求 / 结果，或重新设计同步边界 |
| Lambda 偶发崩溃 | 引用捕获已销毁、裸指针失效、缺失 context | 使用值捕获、正确 context 与明确所有权 |
| disconnect 后仍收到旧结果 | 已排队事件、已执行中的槽 | 增加请求编号或取消检查 |
| `moveToThread` 失败 | 对象存在 parent、调用线程不正确 | 在原线程中准备无 parent 的 Worker，再移动 |
| 退出时提示线程仍在运行 | QThread 已析构但后台尚未结束 | 请求正常退出并 `wait()`，清理 Worker |

记录线程时，可在发射处和槽入口打印：

```cpp
qDebug() << "当前执行线程" << QThread::currentThread()
         << "接收对象所属线程" << receiver->thread();
```

只有两处日志与对象归属一起看，才能判断实际发生了什么。

## 15. 连接方式选择速查

| 需求 | 通常选择 |
| --- | --- |
| 普通同线程对象通知，跨线程时希望自动投递 | Auto，通常省略最后一个参数 |
| 明确要求同步完成，且确认当前线程访问安全 | Direct |
| 同线程延后处理、避免立即进入接收者 | 带 context 的 Queued |
| 后台结果回到 UI 线程 | UI 对象作为 receiver / context，Auto 或明确 Queued |
| 主线程给 Worker 分配任务 | Worker 作为 receiver，通常 Queued |
| 严格受控的跨线程同步操作 | BlockingQueued，先排除死锁与无限等待条件 |
| 防止相同成员函数连接重复建立 | 基础策略叠加 Unique |
| 某个信号只响应一次 | Qt 6 中基础策略叠加 SingleShot |
| Lambda 在重新绑定后只保留一条连接 | 保存句柄，断开旧连接后重建 |
| 多次请求仅接收最新结果 | 连接策略之外，再增加请求编号 / 过期结果检查 |

阅读 `connect` 时可以依次确认四件事：**谁发信号、谁负责回调生命周期、槽在哪里执行、调用什么时候发生。** 对于队列连接，再检查参数保存方式和接收线程的事件循环。

## 16. 验证记录与参考资料

验证日期：2026-10-30。已在 macOS、Qt 6.8.3、C++17、CMake Debug 配置下，直接提取本文的三个完整 `main.cpp` 与通用 CMake 配置，在仓库外的临时目录分别构建运行，均正常退出：

| 验证项 | 结果 |
| --- | --- |
| 示例一：同线程 Auto / Direct / Queued | 日志顺序与预期一致；Auto 和 Queued 的线程断言通过 |
| 示例二：Worker + QThread | 后台处理与主线程接收断言通过；自定义结果成功传递；线程正常退出 |
| 示例三：Unique + 队列 SingleShot | 重复连接被拒绝；成员函数调用 2 次；单次回调仅收到 `10`，调用 1 次 |
| Markdown 检查 | 代码围栏配对、相对文件链接及 README 索引检查通过 |

验证范围是上述完整程序。其他局部片段用于说明相应 API，未逐段组成独立程序运行；Qt 5 兼容说明依据 API 版本边界，未在 Qt 5 环境实测。死锁反例未运行。

进一步阅读：

- [Qt Signals & Slots](https://doc.qt.io/qt-6.8/signalsandslots.html)：信号槽基础与多连接行为。
- [Qt::ConnectionType](https://doc.qt.io/qt-6.8/qt.html#ConnectionType-enum)：全部连接类型与附加标志。
- [QObject](https://doc.qt.io/qt-6.8/qobject.html)：连接重载、线程归属、自动断开与销毁规则。
- [QThread](https://doc.qt.io/qt-6.8/qthread.html)：Worker 模式、线程退出与删除。
- [QMetaType](https://doc.qt.io/qt-6.8/qmetatype.html)：自定义类型的声明与注册。
- [Qt QTimer：基础用法、事件循环与网络超时实战](Qt%20QTimer定时器-基础知识与网络超时实战.md)：对照定时调用、事件循环和业务取消。
