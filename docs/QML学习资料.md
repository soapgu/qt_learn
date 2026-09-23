# Qt Quick/QML 学习资料：从 WPF 与 Android 迁移理解

## 1. 阅读说明

本文面向熟悉 WPF、Caliburn.Micro 和 Android XML/DataBinding 的开发者，以 Qt 6.8.3 和表决系统为主线。WPF 示例使用 C#/XAML/Caliburn.Micro；Android 示例使用 Java/XML DataBinding，并参考 `/Users/guhui/GitLabs/s365-android` 的 `ObservableViewModel`、`MVVMFragment`、`MutableLiveData`、`RecyclerView.Adapter` 和 `ObservableList`。

官方资料：

- [Qt 6.8 Qt Qml](https://doc.qt.io/qt-6.8/qtqml-index.html)
- [Qt 6.8 Qt Quick](https://doc.qt.io/qt-6.8/qtquick-index.html)
- [Qt Quick Model/View](https://doc.qt.io/qt-6.8/qtquick-modelviewsdata-modelview.html)

三套技术解决的是同一类问题：

```text
QML View ⇄ C++ ViewModel → Service → Adapter/设备 SDK
XAML View ⇄ C# ViewModel → Service → 领域对象
XML View ⇄ Android ViewModel → Repository/Service → 平台接口
```

状态通过绑定流向 View，点击等用户意图流向 ViewModel，业务流程继续交给 Service。

### 1.1 术语速查

| 职责 | Qt Quick/QML | WPF/Caliburn.Micro | Android/DataBinding |
| --- | --- | --- | --- |
| 声明界面 | QML | XAML | layout XML |
| 页面状态 | `QObject` ViewModel | `Screen` | `ViewModel`/`AndroidViewModel` |
| 属性通知 | `Q_PROPERTY` + `NOTIFY` | `NotifyOfPropertyChange` | `@Bindable` + `BR`，或 `LiveData` |
| 页面动作 | `Q_INVOKABLE`/slot | Action/`ICommand` | XML lambda 调用方法 |
| 瞬时事件 | signal | event/EventAggregator | `LiveData` 或事件通道 |
| 列表 | `ListView` | `ItemsControl`/`ListBox` | `RecyclerView` |
| 列表数据 | `QAbstractListModel` | `ObservableCollection<T>` | 集合 + Adapter 通知 |
| 单项模板 | delegate | `DataTemplate` | item XML + `ViewHolder` |
| 页面装配 | C++ 组合根 | Bootstrapper/IoC | `ViewModelProvider` |

这些是职责上的近似对应，不表示类型和生命周期完全相同。

### 1.2 两种 Model

**MVVM 业务 Model** 表示业务数据和规则，例如 `VoteSession`、`Voter`、`VoteResult`。

**Qt Model/View 的 Model** 是集合数据源接口。`QAbstractListModel` 向 View 提供行数、角色、数据和增删通知，接近：

- WPF 的 `ObservableCollection<T>` 加条目属性访问；
- Android 的数据集合、`RecyclerView.Adapter` 数据访问和更新通知的组合。

因此 `EventLogModel` 不是整个 MVVM 中的业务 Model，只是日志列表的数据源适配层。

## 2. QML 技术栈与对象树

| 名称 | 作用 | 熟悉概念 |
| --- | --- | --- |
| QML | 声明对象、属性和绑定 | XAML、layout XML |
| Qt Qml | 引擎、类型系统、C++ 集成 | 绑定引擎和生成机制 |
| Qt Quick | 视觉元素、场景图、View | WPF 控件、Android View |
| Quick Controls | Button、Label 等应用控件 | Controls/widgets |
| Quick Layouts | 行、列、网格布局 | Panel/ViewGroup |

```qml
import QtQuick
import QtQuick.Controls
import QtQuick.Layouts

ApplicationWindow {
    width: 720; height: 520; visible: true
    ColumnLayout {
        anchors.fill: parent
        Label { text: qsTr("表决 SDK 最小验证") }
        Button { text: qsTr("启动服务") }
    }
}
```

它对应 WPF 的 `Window → StackPanel → TextBlock/Button`，也对应 Android 的 `LinearLayout → TextView/Button`。三者都是声明式对象树，但所有权不同：QML/C++ 有 QObject 父子关系，WPF 有逻辑树和视觉树，Android View 树受 Activity/Fragment 生命周期管理。

不要把所有页面写进 `Main.qml`。`ServiceStatusPanel.qml`、`ServiceControlPanel.qml`、`EventLogPanel.qml` 等组件分别近似 WPF `UserControl` 和 Android 独立 layout/自定义 View。

顶层表单优先使用 Layout，组件内部简单定位可用 anchors：

```qml
ColumnLayout {
    anchors.fill: parent
    ServiceStatusPanel { Layout.fillWidth: true }
    EventLogPanel { Layout.fillWidth: true; Layout.fillHeight: true }
}
```

不要让同一个 Item 的 anchors 和 Layout 竞争尺寸。这类似 WPF 中混用 Panel 布局与 Canvas 定位，也类似 Android 子 View 必须遵守父 ViewGroup 的 `LayoutParams`。

## 3. 属性和绑定

### 3.1 QML

```qml
Button {
    text: voteViewModel.running ? qsTr("运行中") : qsTr("启动服务")
    enabled: !voteViewModel.running
}
Label { text: voteViewModel.statusText }
```

依赖项改变时表达式会重新计算。不要随后写 `statusLabel.text = "正在启动"`，这会替换原绑定。应调用 ViewModel，让状态再通过绑定返回。

### 3.2 WPF/Caliburn.Micro

```csharp
public bool Running {
    get => running;
    set {
        if (running == value) return;
        running = value;
        NotifyOfPropertyChange();
        NotifyOfPropertyChange(nameof(CanStartServer));
    }
}
public bool CanStartServer => !Running;
```

```xml
<Button Content="启动服务" IsEnabled="{Binding CanStartServer}" />
```

派生属性也需要通知。Caliburn.Micro 还能按 `CanStartServer` 约定控制 `StartServer` 动作。

### 3.3 Android/DataBinding

```java
public final class VoteServerViewModel extends ObservableViewModel {
    private boolean running;

    @Bindable public boolean isRunning() { return running; }

    public void setRunning(boolean value) {
        if (running == value) return;
        running = value;
        notifyPropertyChanged(BR.running);
    }
}
```

```xml
<Button
    android:text="启动服务"
    android:enabled="@{!viewModel.running}" />
```

这与 `s365-android` 中 `StatusViewModel.online` 和 `fragment_status.xml` 的模式相同。`@Bindable` 生成 `BR` 字段，setter 负责通知。

## 4. 动作、信号与事件

QML 可复用组件通过信号报告意图：

```qml
Item {
    signal startRequested()
    Button { text: qsTr("启动"); onClicked: startRequested() }
}

ServiceControlPanel {
    onStartRequested: voteViewModel.startServer()
}
```

监听非父子对象信号可以使用 `Connections`。长期可读取的内容应建模为属性，“控制器刚刚连接”这类一次性通知才适合信号。

WPF/Caliburn.Micro：

```csharp
public void StartServer() => service.Start();
public bool CanStartServer => !Running;
```

```xml
<Button x:Name="StartServer" Content="启动服务" />
<!-- 或 cal:Message.Attach="[Action StartServer]" -->
```

Android/DataBinding：

```xml
<Button android:onClick="@{() -> viewModel.startServer()}" />
```

这与 `s365-android` 中调用 `viewModel.registerFromFiles()` 的方式一致。

不要把 signal、C# event 和 `LiveData` 当成同一个东西。signal 本身不保存历史值；C# event 的生命周期取决于订阅；`LiveData` 保存最近值并只通知活跃 Observer。它们只在“ViewModel 向 View 发出通知”这个职责上相似。

## 5. C++ ViewModel 与生命周期

`Q_OBJECT`、`Q_GADGET`、`Q_ENUM`、`Q_PROPERTY` 与 QML 注册宏的职责和选择规则，参见：[Qt 元对象系统与 QML 类型注册](Qt元对象系统与QML类型注册.md)。

```cpp
class VoteServerViewModel final : public QObject
{
    Q_OBJECT
    Q_PROPERTY(bool running READ running NOTIFY runningChanged)
    Q_PROPERTY(QString statusText READ statusText NOTIFY statusTextChanged)

public:
    Q_INVOKABLE void startServer();
    Q_INVOKABLE void stopServer();

signals:
    void runningChanged();
    void statusTextChanged();
};
```

setter 只在值改变时通知：

```cpp
void VoteServerViewModel::setRunning(bool value)
{
    if (m_running == value) return;
    m_running = value;
    emit runningChanged();
}
```

```text
Q_PROPERTY + NOTIFY
  ≈ INotifyPropertyChanged / NotifyOfPropertyChange
  ≈ @Bindable + BR + notifyPropertyChanged
```

注册类型并由组合根创建：

```cpp
class VoteServerViewModel final : public QObject
{
    Q_OBJECT
    QML_ELEMENT
    QML_UNCREATABLE("由应用程序创建")
};

VoteServerViewModel viewModel(service);
engine.rootContext()->setContextProperty("voteViewModel", &viewModel);
engine.loadFromModule("VoteDevice", "Main");
```

- Qt：C++ 对象必须活到 QML 不再使用它；不要暴露局部临时对象。
- WPF：Bootstrapper/IoC 创建 ViewModel，ViewLocator 找到 View 并设置 `DataContext`；`Screen` 有激活和关闭生命周期。
- Android：项目的 `MVVMFragment` 使用 `ViewModelProvider`、`binding.setVariable` 和 `setLifecycleOwner`。ViewModel 可跨配置变更保留，Fragment View 销毁时 ViewModel 不一定销毁。

普通 QObject 不自动具有 Android `ViewModelStore` 的语义。

## 6. QML 模块

```cmake
find_package(Qt6 6.8 REQUIRED COMPONENTS Core Network Gui Qml Quick QuickControls2)

qt_add_qml_module(vote_app
    URI VoteDevice
    VERSION 1.0
    QML_FILES qml/Main.qml qml/components/EventLogPanel.qml
    SOURCES src/presentation/VoteServerViewModel.cpp
            src/presentation/VoteServerViewModel.h
)
```

它处理资源、类型信息、模块注册和缓存步骤，可类比 WPF XAML 编译/资源注册和 Android Gradle 插件处理 layout/DataBinding 生成的一部分职责。正式部署不要从源码绝对 `file://` 路径加载 QML。

## 7. Model、View 和 Delegate

```text
Model      提供数据、角色和集合通知
View       决定列表、网格、表格或树形排列
Delegate   定义每条数据的显示和交互
```

| 职责 | Qt | WPF | Android |
| --- | --- | --- | --- |
| 列表容器 | `ListView` | `ListBox` | `RecyclerView` |
| 数据集合 | `QAbstractListModel` | `ObservableCollection<T>` | `ObservableList<T>`/`List<T>` |
| 数据桥梁 | Model + View | `ItemsSource` | Adapter |
| 单项模板 | delegate | `DataTemplate` | item XML |
| 项目实例 | View 创建 delegate | ItemsControl 创建容器 | Adapter 创建/绑定 ViewHolder |

### 7.1 先理解一行数据怎样到达界面

假设一条 C++ 日志是：

```cpp
struct EventLogEntry
{
    QString timeText;
    QString message;
};
```

三套框架都要把这两个字段交给条目模板：

```text
Qt       EventLogEntry → data(index, role) → role 名 → delegate 属性
WPF      EventLogItem  → 当前项 DataContext  → 属性绑定
Android  EventLogItem  → Adapter/ViewBinding → item XML variable
```

### 7.2 Qt：role 是 Model 提供给 delegate 的命名字段

`QAbstractListModel` 不把 `EventLogEntry` 对象直接交给 QML，而是让 View 用“第几行、要哪个 role”来取值。

```cpp
class EventLogModel final : public QAbstractListModel
{
    Q_OBJECT

public:
    enum Role {
        TimeTextRole = Qt::UserRole + 1,
        MessageRole
    };

    int rowCount(const QModelIndex &parent = {}) const override
    {
        return parent.isValid() ? 0 : m_entries.size();
    }

    QVariant data(const QModelIndex &index, int role) const override
    {
        if (!index.isValid() || index.row() >= m_entries.size()) {
            return {};
        }

        const EventLogEntry &entry = m_entries.at(index.row());
        switch (role) {
        case TimeTextRole:
            return entry.timeText;
        case MessageRole:
            return entry.message;
        default:
            return {};
        }
    }

    QHash<int, QByteArray> roleNames() const override
    {
        return {
            {TimeTextRole, "timeText"},
            {MessageRole, "message"}
        };
    }

private:
    QList<EventLogEntry> m_entries;
};
```

这里存在两套名字：

| C++ 内部标识 | 暴露给 QML 的 role 名 |
| --- | --- |
| `TimeTextRole` | `timeText` |
| `MessageRole` | `message` |

delegate 再声明同名的接收属性：

```qml
ListView {
    model: eventLogModel

    delegate: RowLayout {
        width: ListView.view.width

        required property string timeText
        required property string message

        Label { text: timeText }
        Label { text: message; Layout.fillWidth: true }
    }
}
```

准确地说，`required property string timeText` 本身不是 role。它是 delegate 的必填属性；因为属性名 `timeText` 与 Model 暴露的 role 名相同，`ListView` 创建 delegate 时会把当前行的 `TimeTextRole` 值传给它。若 Model 没有提供 `timeText`，delegate 创建时会报缺少必填属性。

完整取值链路是：

```text
第 3 行 delegate 需要 timeText
→ ListView 查到 roleNames()[TimeTextRole] == "timeText"
→ 调用 data(index(2, 0), TimeTextRole)
→ 得到 m_entries[2].timeText
→ 赋给 delegate.timeText
→ Label.text 通过绑定显示该值
```

新增数据时必须通知 View 结构将发生变化：

```cpp
void EventLogModel::append(const EventLogEntry &entry)
{
    const int newRow = m_entries.size();
    beginInsertRows({}, newRow, newRow);
    m_entries.append(entry);
    endInsertRows();
}
```

### 7.3 简单列表模型：`modelData` 是当前项的约定名称

`QStringList`、JavaScript 数组和整数模型等简单模型没有 `timeText`、`message` 这样的多个命名 role。对于这类模型，View 会用约定的 `modelData` 向 delegate 提供当前项的值，同时用 `index` 提供当前项的索引。

表决系统的 `VoteServerViewModel` 把 C++ `QStringList` 暴露为 `controllerIps`：

```cpp
Q_PROPERTY(QStringList controllerIps
           READ controllerIps
           NOTIFY controllerIpsChanged)
```

`ControllerListPanel` 将它作为 `ListView` 的模型：

```qml
ListView {
    id: controllerList
    model: root.controllerIps

    delegate: Rectangle {
        required property string modelData

        Label {
            text: modelData
        }
    }
}
```

如果 delegate 还需要当前项的位置，可以另外声明 View 提供的 `index`：

```qml
delegate: Label {
    required property string modelData
    required property int index
    text: qsTr("%1. %2").arg(index + 1).arg(modelData)
}
```

完整传递链路是：

```text
QStringList 中第 2 个字符串
→ ListView 创建第 2 个 delegate
→ 把当前字符串赋给 delegate.modelData
→ Label.text 通过绑定显示 IP
```

`modelData` 不是可以随意改名的局部变量。下面的写法不成立：

```qml
delegate: Rectangle {
    required property string aaa
    Label { text: aaa }
}
```

模型没有向 delegate 提供名为 `aaa` 的属性，所以 `required property string aaa` 无法初始化，delegate 创建时会报缺少必填属性。

如果希望在 delegate 内部使用自定义名称，应显式把它绑定到 `modelData`：

```qml
delegate: Rectangle {
    required property string modelData
    readonly property string aaa: modelData

    Label { text: aaa }
}
```

也可以省略 `required property` 声明，直接使用 delegate 上下文中的 `modelData`：

```qml
delegate: Label {
    text: modelData
}
```

但更推荐显式声明 `required property string modelData`：它能清楚表达 delegate 依赖的模型数据，并让 QML 工具进行类型检查和必填属性检查。

简单模型与 role 模型的区别是：

| Model 形式 | delegate 取值方式 | 名称来源 |
| --- | --- | --- |
| `QStringList` / JavaScript 数组 | `modelData` | QML View 为当前项提供的约定名称 |
| `ListModel` | `ip`、`connected` 等 | `ListElement` 中的字段名 |
| `QAbstractListModel` | `timeText`、`message` 等 | C++ `roleNames()` 返回的 role 名 |

因此，`modelData`、`ip`、`timeText` 都不是 delegate 可以随意改名的变量；它们必须与模型实际提供的名称一致。只有 delegate 自己声明并显式绑定的普通属性，才可以自由命名。

### 7.4 WPF：当前条目直接成为 DataContext

WPF 通常不需要 role 映射，条目是带属性的对象：

```csharp
public sealed class EventLogItem
{
    public string TimeText { get; init; }
    public string Message { get; init; }
}

public ObservableCollection<EventLogItem> EventLogs { get; } = new();
```

```xml
<ListBox ItemsSource="{Binding EventLogs}">
    <ListBox.ItemTemplate>
        <DataTemplate>
            <StackPanel Orientation="Horizontal">
                <TextBlock Text="{Binding TimeText}" />
                <TextBlock Text="{Binding Message}" />
            </StackPanel>
        </DataTemplate>
    </ListBox.ItemTemplate>
</ListBox>
```

`DataTemplate` 的 `DataContext` 就是当前 `EventLogItem`。`ObservableCollection` 通知集合增删；若条目属性还会变化，`EventLogItem` 自身也要发属性变更通知。

### 7.5 Android：Adapter 把当前条目交给 item XML

```xml
<!-- item_event_log.xml -->
<layout>
    <data>
        <variable name="item" type="com.example.EventLogItem" />
    </data>

    <LinearLayout android:orientation="horizontal">
        <TextView android:text="@{item.timeText}" />
        <TextView android:text="@{item.message}" />
    </LinearLayout>
</layout>
```

```java
public void onBindViewHolder(EventLogViewHolder holder, int position) {
    EventLogItem item = items.get(position);
    holder.binding.setItem(item);
    holder.binding.executePendingBindings();
}
```

`s365-android` 的 `ShadowAdapter<T>` 做的正是这种通用绑定：从集合取得当前位置的对象，通过 `binding.setVariable(...)` 交给 item XML，并监听 `ObservableList`。

```text
QAbstractListModel = 数据访问 + role + 集合通知
ObservableCollection<T> = 类型化集合 + 集合通知
RecyclerView = List/ObservableList + Adapter + notifyItem...
```

`beginInsertRows/endInsertRows` 在职责上接近 `CollectionChanged` 和 `notifyItemInserted`。

### 7.6 View 附加能力

- QML `header/footer`：WPF/Android 通常用外层布局，Android 也可用不同 `viewType`。
- QML `section`：WPF 可用 `CollectionViewSource.GroupDescriptions`；Android 通常使用标题条目类型。
- QML `highlight/currentIndex`：接近 WPF `SelectedItem`；RecyclerView 没有统一选择模型，通常由 Adapter/ViewModel 保存。

简单原型可使用整数 model、JavaScript 数组或 QML `ListModel`；正式动态 C++ 数据使用 `QAbstractListModel`。`ObjectModel` 保存已创建的视觉对象，适合少量固定页面，不适合大量日志。

## 8. 表决系统完整数据流

```qml
ColumnLayout {
    Label { text: voteViewModel.statusText }
    RowLayout {
        Button {
            text: qsTr("启动")
            enabled: !voteViewModel.running
            onClicked: voteViewModel.startServer()
        }
        Button {
            text: qsTr("停止")
            enabled: voteViewModel.running
            onClicked: voteViewModel.stopServer()
        }
    }
    ListView {
        model: voteViewModel.eventLogModel
        delegate: Label {
            required property string displayText
            required property int level
            text: displayText
            color: level === EventLogModel.Error ? "#c62828" : "#202124"
        }
        onCountChanged: positionViewAtEnd()
    }
}
```

WPF 使用 `StatusText`、`Running`、`StartServer/StopServer`、`ObservableCollection<EventLogItem>`；Android 使用 `@Bindable` 状态、XML lambda、`ObservableList<EventLogItem>` 和 Adapter。三者的数据流必须一致：

```text
点击启动 → ViewModel 命令 → Service.start
→ Service/设备回调 → 更新 ViewModel 状态与日志
→ 变更通知 → View 自动更新
```

ViewModel 不应在调用 Service 前伪造“已运行”。最终状态由 Service 或设备回调决定。

Mock 后端也遵守同一边界：Mac 注入 `MockVoteServer`，麒麟注入 `VoteServerAdapter`，QML 不出现 `#ifdef` 或供应商类型。这对应 WPF IoC 替换 Service 和 Android 依赖注入替换 Repository。

## 9. 常见错误

1. **在 QML 写业务状态机**：QML 只报告意图，流程交给 ViewModel/Service。
2. **混淆两种 Model**：`VoteResult` 是业务 Model；`EventLogModel` 是集合数据源。
3. **用 JavaScript 数组保存长期数据**：需要动态通知时使用 `QAbstractListModel`。
4. **delegate 直接操作 SDK**：应调用 `voteViewModel.selectVoter(voterId)`。
5. **混用状态和事件**：`running` 是属性；一次性连接通知才可能是 signal。先判断是否需要保存最后值。
6. **忽略生命周期**：Qt 不会替你提供 Android `ViewModelStore`；组合根必须管理所有权。
7. **只在 Mac 验证渲染**：麒麟还要验证 import、X11、字体缩放、图形后端和远程桌面。
8. **所有页面写进 Main.qml**：视觉组件、ViewModel、集合 Model、Service、SDK Adapter 各守边界。

## 10. 渐进练习

1. 创建状态卡片组件，通过属性改变文字和颜色。
2. 绑定 `running`，让启停按钮自动切换可用状态，不直接操作按钮属性。
3. 接入 C++ ViewModel，暴露 `running`、`statusText` 和启停方法。
4. 用 `ListModel` 和 delegate 显示日志，指出它与 `ItemsSource`、Adapter 数据集的对应关系。
5. 换成 `QAbstractListModel`，加入时间、级别、上下文和消息 role；验证逐行新增和颜色显示。
6. Mac 使用 Mock，麒麟使用真实 Adapter；两端共用 QML 和 ViewModel。

## 11. 学完后应能回答

1. MVVM 业务 Model 和 Qt 集合 Model 有什么区别？
2. 为什么 `QAbstractListModel` 不能简单等于 WPF 或 Android 的单个类？
3. 三套属性通知怎样触发界面更新？
4. delegate、`DataTemplate`、item XML + ViewHolder 分别负责什么？
5. 属性、signal、event、`LiveData` 的选择依据是什么？
6. QML、ViewModel、Qt Model、Service、供应商 Adapter 的边界在哪里？

## 12. 后续学习

按顺序学习 Qt Quick Controls 样式、Layouts 与高 DPI、QML 模块化、Qt Quick Test、`QSortFilterProxyModel`、性能分析、国际化和 Linux 部署。

学习 C++ 与 QML 边界时，继续阅读：[Qt 元对象系统与 QML 类型注册](Qt元对象系统与QML类型注册.md)。

始终保持边界：QML 负责展示，ViewModel 负责页面状态与用户意图，Qt Model 负责集合数据源，Service 负责业务流程，Adapter 负责供应商 SDK。
