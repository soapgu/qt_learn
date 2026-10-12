# 如何在 Qt 项目中使用第三方库——入门级教程

> 本文以 caliburn-micro-qt 项目为例，介绍如何在 Qt/CMake 项目中使用 FetchContent 管理第三方依赖。适合没有 C++ 经验的开发者阅读。

---

## 一、Qt 项目的第三方依赖管理工具

在 C++/Qt 项目中引入第三方库，常见的方式有以下几种：

| 工具 | 性质 | 特点 |
|------|------|------|
| **FetchContent** | CMake 内置模块 | 配置时自动下载源码，随项目一起编译，零额外安装 |
| **vcpkg** | 独立包管理器 | 需要安装工具链，支持版本锁定和二进制缓存，适合多依赖管理 |
| **Conan** | 独立包管理器 | 支持源码和二进制包，跨平台能力强，学习曲线较陡 |
| **手动管理** | 无工具 | 下载源码放入项目或手动安装，版本管理困难 |

### 1.1 各工具的适用场景

- **FetchContent**：依赖少（1-3 个）、希望零配置、快速上手的项目
- **vcpkg**：依赖较多（5 个以上）、需要统一管理版本、团队协作项目
- **Conan**：跨平台/跨编译器矩阵、需要预编译二进制包的大型项目

本文重点介绍 **FetchContent**，它是 CMake 官方提供的模块，无需安装任何额外工具，适合入门使用。

---

## 二、FetchContent 是什么？

### 2.1 基本概念

FetchContent 是 CMake 3.11 引入的内置模块，它的作用是：

1. **在 CMake 配置阶段**（运行 `cmake ..` 时）从指定 URL 下载源码
2. **将下载的源码加入项目**，和你自己的代码一起编译
3. **自动处理依赖关系**，让你可以像使用自己的代码一样使用第三方库

### 2.2 工作流程

```
┌─────────────────────────────────────────────────────────────┐
│  开发者运行 cmake ..                                         │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  CMake 读取 CMakeLists.txt                                   │
│  遇到 FetchContent_Declare() 记录要下载的库                   │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  FetchContent_MakeAvailable() 触发下载                        │
│  - 从 URL 下载源码压缩包                                      │
│  - 解压到 _deps 子目录                                        │
│  - 执行第三方库的 CMakeLists.txt                               │
└──────────────────────┬──────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  第三方库编译成静态库/动态库                                    │
│  你的项目通过 target_link_libraries() 链接使用                 │
└─────────────────────────────────────────────────────────────┘
```

### 2.3 与 npm/nuget/Maven 的区别

| 特性 | npm/nuget/Maven | FetchContent |
|------|----------------|--------------|
| 下载时机 | 安装命令（npm install） | CMake 配置时自动下载 |
| 包格式 | 预编译包/二进制 | 源码，需要本地编译 |
| 版本锁定 | package.json/lock 文件 | 通过 URL + Hash 锁定 |
| 依赖解析 | 自动处理依赖树 | 不处理，需手动解决冲突 |
| 缓存 | 全局缓存，多项目共享 | 每个项目独立下载 |
| 工具安装 | 需要安装 Node/NuGet/JDK | CMake 自带，无需额外安装 |

---

## 三、实战：在 caliburn-micro-qt 中使用 FetchContent

### 3.1 项目结构

```
caliburn-micro-qt/
├── CMakeLists.txt                    # 根 CMake 配置
├── cmake/
│   └── ExampleDependencies.cmake     # 依赖管理配置（重点）
├── examples/minimal/
│   ├── CMakeLists.txt                # 示例项目配置
│   └── app/
│       ├── main.cpp
│       └── logging/
│           └── LoggingRuntime.cpp    # 使用 spdlog 的代码
└── third_party/                      # 已移除，改用 FetchContent
```

### 3.2 第一步：创建依赖配置文件

在 `cmake/ExampleDependencies.cmake` 中集中管理所有依赖：

```cmake
include_guard(GLOBAL)
include(FetchContent)

# 兼容 CMake 3.24 之前的版本
if(POLICY CMP0135)
    cmake_policy(SET CMP0135 NEW)
endif()

function(caliburn_fetch_example_dependencies)
    # 这里配置第三方库的编译选项（后面详细解释）
    set(SPDLOG_BUILD_EXAMPLE OFF)
    set(SPDLOG_BUILD_TESTS OFF)
    # ... 更多选项

    # 声明 spdlog 依赖
    FetchContent_Declare(spdlog
        URL https://codeload.github.com/gabime/spdlog/tar.gz/refs/tags/v1.17.0
        URL_HASH SHA256=d8862955c6d74e5846b3f580b1605d2428b11d97a410d86e2fb13e857cd3a744
        TLS_VERIFY ON
    )

    # 声明 Boost.DI 依赖（单头文件库）
    FetchContent_Declare(boost_di
        URL https://raw.githubusercontent.com/boost-ext/di/v1.3.2/include/boost/di.hpp
        URL_HASH SHA256=1680ca33ffa04457edcea5c14346ad6e3138bb34d4fe074a59e13e34b54827
        DOWNLOAD_NO_EXTRACT TRUE
        TLS_VERIFY ON
    )

    # 下载并配置所有依赖
    FetchContent_MakeAvailable(spdlog boost_di)
endfunction()

caliburn_fetch_example_dependencies()
```

### 3.3 第二步：在根 CMakeLists.txt 中引入

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.21)
project(CaliburnMicroQt VERSION 0.1.0 LANGUAGES CXX)

# ... 其他配置 ...

# 引入依赖管理
add_subdirectory(examples/minimal)  # 在子目录中调用依赖配置
```

在 `examples/minimal/CMakeLists.txt` 中：

```cmake
# 引入依赖配置文件
include("${PROJECT_SOURCE_DIR}/cmake/ExampleDependencies.cmake")

# 现在可以使用 spdlog 和 Boost.DI 了
```

### 3.4 第三步：在你的代码中使用

```cpp
// LoggingRuntime.cpp
#include <spdlog/spdlog.h>
#include <spdlog/sinks/rotating_file_sink.h>
#include <spdlog/sinks/stdout_color_sinks.h>

void setupLogging() {
    // 创建控制台输出
    auto consoleSink = std::make_shared<spdlog::sinks::stdout_color_sink_mt>();
    
    // 创建文件输出（轮转日志）
    auto fileSink = std::make_shared<spdlog::sinks::rotating_file_sink_mt>(
        "app.log", 1024 * 1024 * 10, 3);  // 10MB，保留3个文件
    
    // 组合输出
    std::vector<spdlog::sink_ptr> sinks{consoleSink, fileSink};
    auto logger = std::make_shared<spdlog::logger>("app", sinks.begin(), sinks.end());
    
    // 设置为默认日志器
    spdlog::set_default_logger(logger);
    spdlog::set_level(spdlog::level::debug);
}

void logMessage() {
    spdlog::info("应用启动");
    spdlog::debug("调试信息：{}", 42);
    spdlog::warn("警告信息");
}
```

---

## 四、关键配置详解

### 4.1 FetchContent_Declare 参数说明

```cmake
FetchContent_Declare(spdlog
    URL https://codeload.github.com/gabime/spdlog/tar.gz/refs/tags/v1.17.0
    URL_HASH SHA256=d8862955c6d74e5846b3f580b1605d2428b11d97a410d86e2fb13e857cd3a744
    TLS_VERIFY ON
)
```

| 参数 | 作用 | 说明 |
|------|------|------|
| `URL` | 下载地址 | 可以是 tar.gz、zip 或单个文件 |
| `URL_HASH` | 文件校验 | 确保下载内容未被篡改，格式为 `SHA256=哈希值` |
| `TLS_VERIFY ON` | HTTPS 验证 | 确保下载安全 |
| `DOWNLOAD_NO_EXTRACT TRUE` | 不解压 | 用于单头文件库，直接下载不打包的文件 |

### 4.2 编译选项设置

```cmake
set(SPDLOG_BUILD_EXAMPLE OFF)
set(SPDLOG_BUILD_TESTS OFF)
set(SPDLOG_BUILD_BENCH OFF)
```

**这些设置的作用**：

spdlog 的 CMakeLists.txt 中有这样的代码：

```cmake
# spdlog 的 CMakeLists.txt
option(SPDLOG_BUILD_EXAMPLE "Build example" ON)
option(SPDLOG_BUILD_TESTS "Build tests" ON)
```

默认情况下，spdlog 会编译示例和测试。通过设置 `OFF`，我们告诉 spdlog：

- **不要编译示例程序**（节省编译时间）
- **不要编译测试**（我们不需要 spdlog 的测试）
- **只编译库本身**

**原理**：CMake 的 `option()` 会检查是否已有同名变量。如果我们在 `FetchContent_MakeAvailable()` 之前设置这些变量，spdlog 的 CMakeLists.txt 就会使用我们的值。

### 4.3 链接第三方库

```cmake
# examples/minimal/CMakeLists.txt
add_library(CaliburnExampleLogging STATIC
    app/logging/LoggingRuntime.cpp
)

# 链接 spdlog
target_link_libraries(CalurnExampleLogging 
    PUBLIC Qt6::Core 
    PRIVATE spdlog::spdlog  # 使用 spdlog
)
```

**说明**：
- `spdlog::spdlog` 是 spdlog 提供的 CMake 目标名
- `PRIVATE` 表示只有本库内部使用 spdlog，不暴露给使用者
- `PUBLIC` 表示本库和使用者都需要 Qt6::Core

---

## 五、源码下载与编译位置

### 5.1 下载位置

执行 `cmake ..` 后，源码会下载到构建目录的 `_deps` 子目录：

```
build/                          # 构建目录
├── _deps/
│   ├── spdlog-src/             # spdlog 源码
│   │   ├── CMakeLists.txt
│   │   ├── include/
│   │   │   └── spdlog/
│   │   │       ├── spdlog.h
│   │   │       └── ...
│   │   └── src/
│   │       └── spdlog.cpp
│   ├── spdlog-subbuild/        # 下载过程文件
│   ├── spdlog-build/           # spdlog 编译目录
│   ├── boost_di-src/           # Boost.DI 源码（单头文件）
│   │   └── di.hpp
│   └── ...
├── CMakeCache.txt
└── ...
```

### 5.2 编译过程

1. **下载阶段**：CMake 使用 `curl` 或内置下载器获取源码
2. **解压阶段**：自动解压 tar.gz/zip（单头文件跳过此步）
3. **配置阶段**：执行第三方库的 CMakeLists.txt
4. **编译阶段**：运行 `cmake --build .` 时一起编译

### 5.3 头文件查找路径

当你在代码中写：

```cpp
#include <spdlog/spdlog.h>
```

编译器怎么找到这个头文件？

**答案**：spdlog 的 CMakeLists.txt 中有这样的配置：

```cmake
# spdlog 的 CMakeLists.txt
target_include_directories(spdlog
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
)
```

当你执行 `target_link_libraries(YourApp PRIVATE spdlog::spdlog)` 时，CMake 会自动把 `include` 目录加入编译器的头文件搜索路径：

```bash
# 实际编译命令类似：
g++ -I/build/_deps/spdlog-src/include ... your_code.cpp
```

所以你的代码可以直接 `#include <spdlog/spdlog.h>`。

---

## 六、特殊情况：单头文件库（Boost.DI）

### 6.1 什么是单头文件库？

有些 C++ 库只有一个头文件，不需要编译，直接在代码中 `#include` 就能用。Boost.DI 就是这样的库。

### 6.2 配置方式

```cmake
FetchContent_Declare(boost_di
    URL https://raw.githubusercontent.com/boost-ext/di/v1.3.2/include/boost/di.hpp
    URL_HASH SHA256=1680ca33ffa04457edcea5c14346ad6e3138bb34d4fe074a59e13e34b54827
    DOWNLOAD_NO_EXTRACT TRUE  # 关键：不解压，直接下载单文件
    TLS_VERIFY ON
)
FetchContent_MakeAvailable(boost_di)
```

**关键区别**：
- `DOWNLOAD_NO_EXTRACT TRUE`：告诉 CMake 不要解压，因为下载的就是单个文件
- 不需要执行上游的 CMakeLists.txt（单头文件库通常没有）

### 6.3 手动创建 CMake 目标

由于 Boost.DI 没有 CMakeLists.txt，我们需要手动创建：

```cmake
# 校验文件完整性
file(SHA256 "${boost_di_SOURCE_DIR}/di.hpp" boost_di_header_sha256)
if(NOT boost_di_header_sha256 STREQUAL "1680ca33ffa04457edcea5c14346ad6e3138bb34d4fe074a59e13e34b54827")
    message(FATAL_ERROR "Boost.Ext.DI 头文件校验失败")
endif()

# 复制头文件到规范位置
configure_file("${boost_di_SOURCE_DIR}/di.hpp"
    "${boost_di_BINARY_DIR}/include/boost/di.hpp" COPYONLY)

# 创建 INTERFACE 库（不编译，只提供头文件路径）
add_library(CaliburnBoostDI INTERFACE)
add_library(Caliburn::BoostDI ALIAS CaliburnBoostDI)
target_include_directories(CaliburnBoostDI 
    SYSTEM INTERFACE "${boost_di_BINARY_DIR}/include"
)
```

**解释**：
- `INTERFACE` 库：不编译任何源文件，只提供头文件和编译选项
- `ALIAS`：创建别名，使用时可以写 `Caliburn::BoostDI`
- `SYSTEM INTERFACE`：标记为系统头文件，编译器不会对这些头文件发出警告

### 6.4 使用方式

```cmake
# 在你的目标中链接
target_link_libraries(YourApp
    PRIVATE Caliburn::BoostDI
)
```

```cpp
// 在代码中使用
#include <boost/di.hpp>

namespace di = boost::di;

auto injector = di::make_injector();
```

---

## 七、与其他包管理器的对比

### 7.1 与 npm 对比

| 方面 | npm | FetchContent |
|------|-----|--------------|
| 声明文件 | package.json | CMakeLists.txt |
| 安装命令 | `npm install` | `cmake ..`（自动） |
| 版本锁定 | package-lock.json | URL + SHA256 |
| 包格式 | 预编译 JS 或源码 | 源码，需编译 |
| 全局缓存 | ~/.npm | 无，每项目独立 |
| 依赖解析 | 自动处理嵌套依赖 | 不处理，需手动 |

**npm 示例**：
```json
// package.json
{
  "dependencies": {
    "lodash": "^4.17.21"
  }
}
```

**FetchContent 对应**：
```cmake
FetchContent_Declare(lodash
    URL https://github.com/lodash/lodash/archive/refs/tags/4.17.21.tar.gz
    URL_HASH SHA256=...
)
```

### 7.2 与 NuGet 对比

| 方面 | NuGet | FetchContent |
|------|-------|--------------|
| 声明文件 | packages.config 或 PackageReference | CMakeLists.txt |
| 安装命令 | `nuget restore` 或 VS 自动 | `cmake ..`（自动） |
| 包格式 | 预编译二进制（nupkg） | 源码，需编译 |
| 版本管理 | 支持浮动版本 | 固定版本（URL 锁定） |
| 离线支持 | 本地缓存 | 需手动缓存源码包 |

### 7.3 与 Maven 对比

| 方面 | Maven | FetchContent |
|------|-------|--------------|
| 声明文件 | pom.xml | CMakeLists.txt |
| 安装命令 | `mvn compile`（自动下载） | `cmake ..`（自动） |
| 仓库 | Maven Central | 任意 URL |
| 依赖解析 | 自动处理传递依赖 | 不处理 |
| 构建生命周期 | 标准化（compile/test/package） | 依赖库自己定义 |

---

## 八、常见问题

### 8.1 下载失败怎么办？

**问题**：网络问题导致下载失败

**解决**：
1. 使用国内镜像（如 GitHub 加速站）
2. 手动下载放到 CMake 缓存目录
3. 使用 `FETCHCONTENT_FULLY_DISCONNECTED` 模式

```cmake
# 离线模式：使用已下载的源码，不再尝试下载
set(FETCHCONTENT_FULLY_DISCONNECTED ON CACHE BOOL "" FORCE)
```

### 8.2 如何查看下载进度？

CMake 默认不显示下载进度。可以设置：

```bash
cmake .. --log-level=VERBOSE
```

或修改 CMakeLists.txt：

```cmake
set(FETCHCONTENT_QUIET OFF)  # 显示下载信息
```

### 8.3 依赖版本冲突怎么办？

**问题**：A 库依赖 spdlog 1.11，B 库依赖 spdlog 1.12

**解决**：
1. 统一使用一个版本（通常是较新的）
2. 使用 `FetchContent_Declare` 的 `OVERRIDE` 选项
3. 迁移到 vcpkg/Conan，让包管理器处理

### 8.4 如何加速重复构建？

FetchContent 下载的源码会保留在 `_deps` 目录，重复运行 `cmake ..` 不会重新下载。

如需清理重新下载：

```bash
rm -rf build/_deps
cmake ..
```

---

## 九、总结

### 9.1 FetchContent 的优缺点

**优点**：
- 零安装：CMake 自带，无需额外工具
- 零配置：不需要设置工具链或环境变量
- 版本锁定：通过 URL + Hash 确保可重复构建
- 集成简单：几行 CMake 代码即可使用

**缺点**：
- 每次构建都要编译：没有预编译包
- 不处理依赖冲突：多个库依赖同一依赖的不同版本时需手动处理
- 无全局缓存：每个项目独立下载
- 网络依赖：首次配置需要网络

### 9.2 何时迁移到 vcpkg？

当出现以下情况时，建议考虑迁移到 vcpkg：

1. 依赖数量超过 5 个
2. 需要处理复杂的依赖关系
3. 团队多人协作，需要统一环境
4. 需要提供预编译包给最终用户
5. 编译时间过长，需要二进制缓存

### 9.3 快速上手清单

- [ ] 创建 `cmake/ExampleDependencies.cmake` 文件
- [ ] 使用 `FetchContent_Declare` 声明依赖
- [ ] 调用 `FetchContent_MakeAvailable` 下载并配置
- [ ] 在目标中使用 `target_link_libraries` 链接
- [ ] 在代码中 `#include` 头文件
- [ ] 运行 `cmake ..` 验证配置
- [ ] 运行 `cmake --build .` 编译项目

---

## 附录：完整示例代码

### cmake/ExampleDependencies.cmake

```cmake
include_guard(GLOBAL)
include(FetchContent)

if(POLICY CMP0135)
    cmake_policy(SET CMP0135 NEW)
endif()

function(caliburn_fetch_example_dependencies)
    # 设置编译选项
    set(CMAKE_POLICY_DEFAULT_CMP0077 NEW)
    set(BUILD_SHARED_LIBS OFF)
    set(SPDLOG_BUILD_EXAMPLE OFF)
    set(SPDLOG_BUILD_TESTS OFF)
    set(SPDLOG_BUILD_BENCH OFF)

    # 声明 spdlog
    FetchContent_Declare(spdlog
        URL https://codeload.github.com/gabime/spdlog/tar.gz/refs/tags/v1.17.0
        URL_HASH SHA256=d8862955c6d74e5846b3f580b1605d2428b11d97a410d86e2fb13e857cd3a744
        TLS_VERIFY ON
    )

    # 声明 Boost.DI（单头文件）
    FetchContent_Declare(boost_di
        URL https://raw.githubusercontent.com/boost-ext/di/v1.3.2/include/boost/di.hpp
        URL_HASH SHA256=1680ca33ffa04457edcea5c14346ad6e3138bb34d4fe074a59e13e34b54827
        DOWNLOAD_NO_EXTRACT TRUE
        TLS_VERIFY ON
    )

    # 下载并配置
    FetchContent_MakeAvailable(spdlog boost_di)

    # 为 Boost.DI 创建 CMake 目标
    configure_file("${boost_di_SOURCE_DIR}/di.hpp"
        "${boost_di_BINARY_DIR}/include/boost/di.hpp" COPYONLY)
    add_library(CaliburnBoostDI INTERFACE)
    add_library(Caliburn::BoostDI ALIAS CaliburnBoostDI)
    target_include_directories(CaliburnBoostDI 
        SYSTEM INTERFACE "${boost_di_BINARY_DIR}/include"
    )
endfunction()

caliburn_fetch_example_dependencies()
```

### 使用方式

```cmake
# 在你的 CMakeLists.txt 中
include("${PROJECT_SOURCE_DIR}/cmake/ExampleDependencies.cmake")

add_executable(MyApp main.cpp)
target_link_libraries(MyApp 
    PRIVATE spdlog::spdlog
    PRIVATE Caliburn::BoostDI
)
```

```cpp
// main.cpp
#include <spdlog/spdlog.h>
#include <boost/di.hpp>

int main() {
    spdlog::info("Hello, FetchContent!");
    return 0;
}
```

---

**参考链接**：
- [CMake FetchContent 官方文档](https://cmake.org/cmake/help/latest/module/FetchContent.html)
- [spdlog 官方仓库](https://github.com/gabime/spdlog)
- [Boost.DI 官方仓库](https://github.com/boost-ext/di)
