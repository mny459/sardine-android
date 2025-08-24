# 重构计划：将 Sardine-Android 迁移到 Kotlin Multiplatform (KMP) 和 Ktor

本文档概述了将 `sardine-android` 库从当前的 Java/OkHttp 实现重构为使用 Ktor 作为 HTTP 客户端的现代化 Kotlin Multiplatform (KMP) 库的计划。

**目标：** 创建一个用于 WebDAV 操作的 KMP 库，该库可以跨多个平台（Android、iOS、JVM 等）共享。

**关键技术：**
*   **Kotlin：** 主要编程语言。
*   **Kotlin Multiplatform (KMP)：** 用于跨平台共享代码。
*   **Ktor：** 作为处理 WebDAV 请求的 HTTP 客户端。
*   **kotlinx.coroutines：** 用于异步操作。
*   **kotlinx.serialization：** 用于 WebDAV 模型对象的 XML 解析和序列化。

---

## 重构阶段

### 第一阶段：项目设置和初始配置

1.  **创建 KMP 模块结构：**
    *   设置一个新的 Gradle 模块，采用 KMP 结构。
    *   定义源集（source sets）：`commonMain`、`androidMain`、`jvmMain`、`iosMain`。
    *   使用必要的 KMP 插件 (`org.jetbrains.kotlin.multiplatform`) 配置 `build.gradle.kts`。

2.  **添加依赖：**
    *   **Ktor：**
        *   在 `commonMain` 中添加 `ktor-client-core`。
        *   在 `commonMain` 中添加 `ktor-client-content-negotiation` 和 `ktor-serialization-kotlinx-xml`。
        *   平台特定的引擎：`androidMain` 使用 `ktor-client-okhttp`，`iosMain` 使用 `ktor-client-darwin`，`jvmMain` 使用 `ktor-client-cio`。
    *   **kotlinx.coroutines：** 在 `commonMain` 中添加 `kotlinx-coroutines-core`。
    *   **kotlinx.serialization：** 在 `commonMain` 中添加 `kotlinx-serialization-core` 和 `kotlinx-serialization-xml`。
    *   **Okio：** 如果需要，使用与 KMP 兼容的 Okio 进行 I/O 操作。
    *   **测试：** 在 `commonTest` 中使用 `kotlin.test` 进行多平台测试。

### 第二阶段：模型和 API 迁移

1.  **迁移数据模型：**
    *   将 `com.thegrizzlylabs.sardineandroid.model` 包中的所有 Java 类转换为 Kotlin 的 `data class`。
    *   将新的 data class 放置在 `commonMain` 源集中。
    *   使用 `@Serializable` 注解它们，以便 `kotlinx.serialization` 进行处理。这将取代当前的 `simple-xml` 依赖。

2.  **重新定义 `Sardine` 接口：**
    *   将 `Sardine.java` 接口转换为 `commonMain` 中的 Kotlin `interface`。
    *   使用 `suspend` 函数替换阻塞方法。
    *   使用 Kotlin 类型（例如，`List`、`Map`，`InputStream` 将被 Ktor 的 `ByteReadChannel` 或 `ByteArray` 替代）。
    *   示例：`fun list(url: String): List<DavResource>` 变为 `suspend fun list(url: String): List<DavResource>`。

### 第三阶段：使用 Ktor 实现

1.  **创建基于 Ktor 的 Sardine 实现：**
    *   在 `commonMain` 中创建一个新类 `KtorSardine`，实现新的 `Sardine` 接口。
    *   使用 `ContentNegotiation` 和 `XML` 插件初始化一个 Ktor `HttpClient`。
    *   使用 Ktor 的 `Auth` 功能（例如 `BasicAuth`）实现身份验证。

2.  **实现 WebDAV 方法：**
    *   使用 Ktor 的请求构建器重新实现所有 WebDAV 方法（`PROPFIND`、`PROPPATCH`、`MKCOL`、`GET`、`PUT`、`DELETE`、`COPY`、`MOVE`、`LOCK`、`UNLOCK`、`REPORT`）。
    *   对 WebDAV 特定的动词使用自定义 HTTP 方法（例如 `HttpMethod("PROPFIND")`）。
    *   使用 `kotlinx.serialization` 处理请求体（XML 序列化）和响应体（XML 反序列化）。
    *   用 Ktor 的响应验证和处理逻辑替换现有的 `ResponseHandler` 类。

3.  **使用 `expect`/`actual` 处理平台特定功能：**
    *   对于像文件系统访问 (`java.io.File`) 这样的功能，在 `commonMain` 中创建 `expect` 声明，并为每个平台（`androidMain`、`jvmMain` 等）提供 `actual` 实现。Okio 在这方面会很有帮助。

### 第四阶段：测试和验证

1.  **迁移单元测试：**
    *   将现有的 JUnit 测试转换为 `kotlin.test`。
    *   将它们放在 `commonTest` 中，以便在所有平台上运行。
    *   模拟 Ktor `HttpClient` 来测试业务逻辑，而无需发出真实的网络请求。

2.  **更新/创建集成测试：**
    *   调整 `FunctionalSardineTest` 和 `IntegrationTest`，使其适用于新的基于 Ktor 的实现。
    *   这些测试应该针对一个真实的 WebDAV 服务器运行，以确保兼容性。

### 第五阶段：文档和发布

1.  **更新 `README.md`：**
    *   添加在不同平台上设置 KMP 库的说明。
    *   提供使用协程和新 API 的示例代码。
    *   记录新的依赖项。

2.  **发布：**
    *   配置 Gradle 将 KMP 库发布到 Maven Central 或其他仓库。

---

## 详细任务清单

- [ ] **第一阶段：项目设置**
    - [ ] 初始化 KMP Gradle 模块。
    - [ ] 配置 `build.gradle.kts` 的插件和源集。
    - [ ] 添加 Ktor、coroutines 和 serialization 的依赖。
    - [ ] 添加测试依赖。

- [ ] **第二阶段：模型和 API 迁移**
    - [ ] 将 `model` 包中的所有类转换为带 `@Serializable` 的 Kotlin data class。
    - [ ] 将 `Sardine.java` 转换为带 `suspend` 函数的 Kotlin 接口。
    - [ ] 将 `DavResource`、`DavAcl` 等转换为 Kotlin 类。

- [ ] **第三阶段：实现**
    - [ ] 创建 `KtorSardine` 类。
    - [ ] 配置 Ktor `HttpClient`，启用 XML 序列化和身份验证。
    *   [ ] 实现 `list` 和 `propfind` (`PROPFIND` 方法)。
    *   [ ] 实现 `patch` (`PROPPATCH` 方法)。
    *   [ ] 实现 `get` (`GET` 方法)。
    *   [ ] 实现 `put` (`PUT` 方法)。
    *   [ ] 实现 `delete` (`DELETE` 方法)。
    *   [ ] 实现 `createDirectory` (`MKCOL` 方法)。
    *   [ ] 实现 `move` (`MOVE` 方法)。
    *   [ ] 实现 `copy` (`COPY` 方法)。
    *   [ ] 实现 `exists` (`HEAD` 方法)。
    *   [ ] 实现 `lock`、`refreshLock`、`unlock` (`LOCK`/`UNLOCK` 方法)。
    *   [ ] 实现 `report` (`REPORT` 方法)。
    - [ ] 为文件处理创建 `expect`/`actual`。

- [ ] **第四阶段：测试**
    - [ ] 迁移 `SardineUtilTest`、`SardineExceptionTest` 等的单元测试。
    - [ ] 更新集成测试以使用新的 API。
    - [ ] 搭建一个带 WebDAV 服务器的测试环境。

- [ ] **第五阶段：文档与发布**
    - [ ] 更新 `README.md`，包含新 API 的用法和设置说明。
    - [ ] 为现有用户编写迁移指南。
    - [ ] 配置并测试发布流程。