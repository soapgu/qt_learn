# Qt 学习资料

本目录整理了 Qt 6、Qt Quick 和 QML 相关的学习文档，包括界面开发、数据绑定、事件通信、元对象系统与环境搭建。

## QML

- [Qt Quick/QML 学习资料](docs/QML学习资料.md)：从 WPF 与 Android 开发经验出发，系统理解 QML 技术栈、对象树和常用组件。
- [Qt QML Signal 与分层事件流](docs/Qt%20QML%20Signal与分层事件流.md)：讲解 QML 自定义信号、控件事件与分层通信。
- [Qt Q_PROPERTY 与跨平台双向绑定对照](docs/Qt%20Q_PROPERTY与跨平台双向绑定对照.md)：对照 WPF 和 Android MVVM，理解 Qt 属性通知与双向绑定。

## Qt 核心机制

- [Qt 元对象系统与 QML 类型注册](docs/Qt元对象系统与QML类型注册.md)：理解 `Q_OBJECT`、信号槽、反射能力与 C++ 类型暴露方式。

## 环境搭建

- [银河麒麟 Qt 6.8.3 环境安装操作手册](docs/银河麒麟Qt6环境安装操作手册.md)：在银河麒麟桌面操作系统上安装 Qt 6.8.3、Qt Creator 并验证工程构建。

## 跨平台实践

- [Mac 与银河麒麟 Qt 5 跨平台开发实践](docs/Mac与银河麒麟Qt5跨平台开发实践.md)：以 Qt 5.12 为兼容基线，记录 Apple Silicon Mac 开发、麒麟原生构建、SSH 维护、GUI 验证与常见故障排查。

## 目录结构

```text
.
├── README.md
└── docs
    ├── *.md
    └── assets
        └── *.png
```
