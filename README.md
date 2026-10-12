# Qt 学习资料

本目录整理了 Qt 6、Qt Quick 和 QML 相关的学习文档，包括界面开发、数据绑定、事件通信、元对象系统与环境搭建。

## QML

- [Qt Quick/QML 学习资料](docs/QML学习资料.md)：从 WPF 与 Android 开发经验出发，系统理解 QML 技术栈、对象树和常用组件。
- [Qt QML Signal 与分层事件流](docs/Qt%20QML%20Signal与分层事件流.md)：讲解 QML 自定义信号、控件事件与分层通信。
- [Qt Q_PROPERTY 与跨平台双向绑定对照](docs/Qt%20Q_PROPERTY与跨平台双向绑定对照.md)：对照 WPF 和 Android MVVM，理解 Qt 属性通知与双向绑定。
- [Qt QML 附加属性与 WPF 对照](docs/Qt%20QML附加属性与WPF对照.md)：理解附加属性、附加信号和实例状态，并对照 Qt C++ 与 WPF 的自定义实现。

## Qt 核心机制

- [Qt 元对象系统与 QML 类型注册](docs/Qt元对象系统与QML类型注册.md)：理解 `Q_OBJECT`、信号槽、反射能力与 C++ 类型暴露方式。
- [Qt C++ 接口类与纯虚函数](docs/Qt%20C++接口类与纯虚函数.md)：拆解纯虚常量成员函数、`Q_DECLARE_INTERFACE`/`Q_INTERFACES` 机制与虚析构函数的三种写法。
- [Qt Q_INVOKABLE 与跨平台命令调用对照](docs/Qt%20Q_INVOKABLE与跨平台命令调用对照.md)：梳理 C++ 方法暴露、动态调用与异步边界，对照 WPF、Caliburn.Micro 和 Android 的界面命令。

## 工程实践

- [C++ 静态库与动态库——基础知识与项目实战](docs/C++静态库与动态库-基础知识与项目实战.md)：从编译、链接与加载入门，结合 spdlog 和表决 SDK 讲解静态库、动态库、CMake 关联和实际调用流程。
- [如何在 Qt 项目中使用第三方库——入门级教程](docs/如何在Qt项目中使用第三方库-入门级教程.md)：以 caliburn-micro-qt 为例，介绍如何使用 CMake FetchContent 管理第三方依赖（spdlog、Boost.DI）。

## 环境搭建

- [银河麒麟 Qt 6.8.3 环境安装操作手册](docs/银河麒麟Qt6环境安装操作手册.md)：在银河麒麟桌面操作系统上安装 Qt 6.8.3、Qt Creator 并验证工程构建。

## 跨平台实践

- [Mac 与银河麒麟 Qt 5 跨平台开发实践](docs/Mac与银河麒麟Qt5跨平台开发实践.md)：以 Qt 5.12 为兼容基线，记录 Apple Silicon Mac 开发、麒麟原生构建、SSH 维护、GUI 验证与常见故障排查。

## 心得体会

- [0经验开发 QT——序章](docs/体会/0经验开发QT-序章.md)：从 WPF/Android 经验出发，分享 AI Agent 辅助与跨系统经验迁移的 Qt 学习路径。

## 目录结构

```text
.
├── README.md
└── docs
    ├── *.md
    ├── 体会
    │   └── *.md
    └── assets
        └── *.png
```
