# Qt 学习笔记与 MVVM 实践

记录从 WPF / Android 开发经验出发学习 Qt 的过程，整理 Qt Quick / QML、C++、元对象系统、MVVM、网络请求与跨平台开发中的知识和实践。

文档结合表决 SDK、贪吃蛇练习、每日一图与 `caliburn-micro-qt` 框架开发展开，通过具体问题理解 Qt 与其他客户端技术的异同。本仓库以学习文档为主，示例代码和项目背景见各篇正文。

多数内容围绕 Qt 6 展开，另有 Qt 5.12 跨平台实践；具体版本和验证范围以各篇说明为准。

## 建议阅读顺序

1. **了解学习背景**：从「心得体会」中的序章开始，了解从已有客户端经验迁移到 Qt 的思路。
2. **建立界面与数据流概念**：先读 QML 学习资料，再学习属性绑定、Signal 与 C++ 方法调用。
3. **补齐底层与工程知识**：按需阅读元对象系统、C++ 接口、第三方依赖以及静态库与动态库。
4. **结合项目深入实践**：阅读 HTTP、定时器、环境搭建与排障记录，再结合框架开发体会理解 MVVM 的设计取舍。

## 文档索引

### Qt Quick / QML 与 MVVM

- [Qt Quick/QML 学习资料](docs/QML学习资料.md)：从 WPF 与 Android 开发经验出发，系统理解 QML 技术栈、对象树和常用组件。
- [Qt QML Signal 与分层事件流](docs/Qt%20QML%20Signal与分层事件流.md)：讲解 QML 自定义信号、控件事件与分层通信。
- [Qt Q_PROPERTY 与跨平台双向绑定对照](docs/Qt%20Q_PROPERTY与跨平台双向绑定对照.md)：对照 WPF 和 Android MVVM，理解 Qt 属性通知与双向绑定。
- [Qt Q_INVOKABLE 与跨平台命令调用对照](docs/Qt%20Q_INVOKABLE与跨平台命令调用对照.md)：梳理 C++ 方法暴露、动态调用与异步边界，对照 WPF、Caliburn.Micro 和 Android 的界面命令。
- [Qt QML 附加属性与 WPF 对照](docs/Qt%20QML附加属性与WPF对照.md)：理解附加属性、附加信号和实例状态，并对照 Qt C++ 与 WPF 的自定义实现。

### Qt 元对象系统与 C++ 基础

- [Qt 元对象系统与 QML 类型注册](docs/Qt元对象系统与QML类型注册.md)：理解 `Q_OBJECT`、信号槽、反射能力与 C++ 类型暴露方式。
- [Qt connect 信号槽连接方式详解](docs/Qt%20connect信号槽连接方式详解.md)：系统讲解连接语法、全部连接类型、队列与跨线程通信、参数传递、连接生命周期及可运行示例。
- [Qt 元对象注解宏详解](docs/Qt%20元对象注解宏详解.md)：讲解 Q_OBJECT、Q_PROPERTY、QML_ELEMENT 等宏的本质、工作机制与 .NET/Java 对比。
- [Qt C++ 接口类与纯虚函数](docs/Qt%20C++接口类与纯虚函数.md)：拆解纯虚常量成员函数、`Q_DECLARE_INTERFACE`/`Q_INTERFACES` 机制与虚析构函数的三种写法。
- [C++ 友元类与友元函数](docs/C++友元类与友元函数.md)：讲解 `friend` 语法、访问控制、使用场景与 Qt 元对象系统的关系。
- [Qt QML_SINGLETON 与 C++ 静态方法](docs/Qt%20QML_SINGLETON与C++静态方法.md)：区分 QML 单例对象、成员方法与静态入口，理解每引擎实例和进程共享配置。

### 网络请求与异步处理

- [Qt 原生 HTTP 实战——从 JSON 请求到图片下载](docs/Qt%20原生%20HTTP%20实战——从%20JSON%20请求到图片下载.md)：结合每日一图项目，介绍 `QNetworkAccessManager`、`QRestAccessManager`、JSON 解析、图片下载，以及取消、超时和连续刷新时的结果管理。
- [Qt QTimer：基础用法、事件循环与网络超时实战](docs/Qt%20QTimer定时器-基础知识与网络超时实战.md)：讲解重复与单次定时器、事件循环、回调生命周期、线程归属，以及首次刷新、请求总时限和防抖等用法。

### 工程实践与依赖管理

- [C++ 静态库与动态库——基础知识与项目实战](docs/C++静态库与动态库-基础知识与项目实战.md)：从编译、链接与加载入门，结合 spdlog 和表决 SDK 讲解静态库、动态库、CMake 关联和实际调用流程。
- [如何在 Qt 项目中使用第三方库——入门级教程](docs/如何在Qt项目中使用第三方库-入门级教程.md)：以 caliburn-micro-qt 为例，介绍如何使用 CMake FetchContent 管理第三方依赖（spdlog、Boost.DI）。

### 环境搭建与跨平台开发

- [银河麒麟 Qt 6.8.3 环境安装操作手册](docs/银河麒麟Qt6环境安装操作手册.md)：在银河麒麟桌面操作系统上安装 Qt 6.8.3、Qt Creator 并验证工程构建。

- [Mac 与银河麒麟 Qt 5 跨平台开发实践](docs/Mac与银河麒麟Qt5跨平台开发实践.md)：以 Qt 5.12 为兼容基线，记录 Apple Silicon Mac 开发、麒麟原生构建、SSH 维护、GUI 验证与常见故障排查。

### 调试与问题排查

- [Qt Creator 日志颜色不生效的诊断与修复](docs/Qt%20Creator日志颜色不生效的诊断与修复.md)：记录 spdlog 日志颜色在 Qt Creator 中不生效的原因、颜色模式与运行环境设置，以及修复验证过程。

### 心得体会与框架开发

- [0经验开发 QT——序章](docs/体会/0经验开发QT-序章.md)：从 WPF/Android 经验出发，分享 AI Agent 辅助与跨系统经验迁移的 Qt 学习路径。
- [从零开始开发 Qt 版本的 CaliburnMicro（MVVM）框架](docs/体会/从零开始开发Qt版本的CaliburnMicro%28MVVM%29框架.md) **（写作中）**：记录从贪吃蛇练习中的轻量内置 MVVM 方案，到独立 Qt 版 Caliburn.Micro 框架的演变；目前写到框架选型、Boost.DI / spdlog 配套依赖，以及 Qt 对象生命周期等平台差异，后续内容待续。

## 目录结构

```text
.
├── README.md          # 仓库介绍与文档索引
└── docs
    ├── *.md           # 技术专题、工程实践与排障记录
    ├── 体会
    │   └── *.md       # 学习体会与框架开发记录
    └── assets
        └── *.png      # 文档插图与验证截图
```
