---
title: 将 Remold 接入 NeoForge 实现热重载
date: 2026-11-11
tags: [指南]
categories: [技术]
description: 本文讲解如何将 Remold 接入到 NeoForge 以及常见错误排查
---

## 为什么要热重载

Minecraft 模组开发最消耗时间的不是写代码，而是验证循环：

改一个数值 → 关游戏 → `gradlew runClient` → 等加载 → 进世界 → 跑到那个场景 → 发现改错了 → 再来一遍。

一次循环动辄两分钟，其中 95% 的时间在等。Remold 解决的就是这一环：它把 `build/classes` 当输入，文件一变就把新字节码原地换进正在运行的 JVM。

它比我以前用的 JVM HotSwap / Spring Loaded 强在几点：

- **支持结构变更**：加字段、加方法、改签名，不只是改方法体（需要 JBR 的 DCEVM 增强重定义）；
- **存量对象补初值**：新加的字段在老对象里不是 `0`/`null`，会被补上初始值（只补零值，不覆盖你手动改过的状态）；
- **匿名类/lambda 按结构指纹对齐**，不会热更几次之后行为串到别的匿名类上；
- **重定义前重新织入 Mixin**，`@Inject` 不会丢；
- 删除字段/改类型而堆里还有存活实例时，**拒绝重定义**而不是把内存改坏。

## Remold 的三种接入方式

上游 [Remold](https://github.com/I-hope1/Remold) 内 README 给了三条路：

| 方式 | 适合 | 说明 |
| --- | --- | --- |
| Gradle 脚本插件 + `runRemold` | 普通 Java / Application 工程 | 自动增参、生成 IDEA 运行配置 |
| IDEA 原生运行配置 | 用断点调试、不占 Gradle | 插件生成 `Run_with_Remold` |
| 手动 `JavaExec` / 命令行 | 特殊入口（如 Minecraft） | `-javaagent` |

**我们的工程不适合第一条** NeoForge 的 dev 启动不是普通 `main`：

```
cpw.mods.bootstraplauncher.BootstrapLauncher
--launchTarget forgeclientdev
--version 21.1.235
--assetIndex 17
...
```

而且 ModDevGradle 会在 `build/moddev/` 下生成一堆参数文件（`clientRunVmArgs.txt`、`clientRunProgramArgs.txt`、`clientLegacyClasspath.txt`），`runRemold` 拼不出这些。

所以正确做法是：**把 agent 织进 ModDevGradle 自带的 `runClient` / `runServer` 任务**，让它们多挂几个 JVM 参数。

---

## 接入步骤

### 准备jar拓展文件

从 [Releases](https://github.com/I-hope1/Remold/releases) 下载 `remold-agent.jar` 和 `remold-plugin-forge.jar`，放到工程的 `libs/remold/`。

### 创建 `gradle/remold.gradle`

```groovy
// gradle/remold.gradle
def remoldEnabled = (findProperty('remold.enabled') ?: 'true').toBoolean()
def agentJar     = file(findProperty('remold.agentJar') ?: 'libs/remold/remold-agent.jar')
def pluginJar    = file(findProperty('remold.forgePluginJar') ?: 'libs/remold/remold-plugin-forge.jar')
def fileShakeMs  = findProperty('remold.fileShakeMs') ?: '1200'

def classDirs = [
    file('build/classes/java/main').absolutePath,
    file('build/resources/main').absolutePath,
]

if (remoldEnabled) {
    ['runClient', 'runServer', 'runData'].each { taskName ->
        tasks.matching { it.name == taskName }.configureEach { t ->
            t.doFirst {
                t.jvmArgs(
                    '-XX:+IgnoreUnrecognizedVMOptions',
                    '-XX:+AllowEnhancedClassRedefinition',
                    "-javaagent:${agentJar.absolutePath}=${classDirs.join(';')}",
                    '-Dremold.plugins=forge',
                    "-Dremold.plugin.jars=${pluginJar.absolutePath}",
                    '-Dremold.build_mode=auto',
                    "-Dremold.file_shake_ms=${fileShakeMs}",
                    '--add-exports=java.base/jdk.internal.misc=ALL-UNNAMED',
                    '--add-exports=java.base/jdk.internal.reflect=ALL-UNNAMED',
                    '--add-exports=java.base/jdk.internal.loader=ALL-UNNAMED',
                    '--enable-native-access=ALL-UNNAMED',
                )
            }
        }
    }
}
```

`build.gradle` 里需要添加 `apply from: 'gradle/remold.gradle'`。

> 监控目录必须是**绝对路径**、`;` 分隔，并且同时包含 `classes` 和 `resources`。资源文件（json/贴图/lang）不走 `compileJava`，但 agent 也盯着 `build/resources/main`。

### 配 JBR

结构变更需要 JBR 17/21/25+ 的增强重定义。`gradle.properties` 内添加绝对路径：

```properties
remold.jbr=you jbr absolute path
```

**若使用环境变量 `JBR_HOME`**，须使用 `gradlew --stop` 杀掉守护进程才会重新读

如果暂时不想给 client 换 JVM（如兼容性等问题），可以只给服务端换：

```properties
remold.jbr.scope=server
```

---

## 验证

### 启动日志

Remold 的 banner 走 **stdout**，所以只能在 **Gradle 控制台**看

```
[Remold] Structural HotSwap Supported: true
JVM identified as JetBrains s.r.o. OpenJDK 64-Bit Server VM 21.0.9+1-b1038.91
```

看到 `false` 或者 `Eclipse Adoptium` 就说明 JBR 没吃到，见第六节。

### 改方法体

任意更改、增加一个字段保存应出现类似于以下内容

```
[Remold] [DIFF] com.example.YourClass:
 *Modified: [yourMethod()V]
[Remold] HotSwap successful: 1 classes redefined.
```

> 注意可能伴随一行 `Field init patch skipped ... use @RemoldReinit(OVERWRITE)`：新字段的初始化写入被保守跳过（因为不知道原对象的状态该不该覆盖），这是设计如此，不是 bug。

### 删除与新增类

直接新建一个自定义类随后应用保存

或者 `git checkout` 还原再编译，应看到 `-Removed: [...]` + `HotSwap successful`

---

## 全自动热更

由于 **Remold** 仅识别 `.class`，所以只改源码不编译，它什么都不会发生

### 命令行 Gradle 持续构建

```bat
:: hot.bat
@echo off
cd /d "%~dp0"
start "remold-watch" cmd /k "timeout /t 25 >nul & gradlew compileJava -t --offline --console=plain"
gradlew runClient --offline
pause
```

`-t` 是 Gradle 的持续构建：持续监测 `src/main/java`，保存增量编译

使用该命令会额外打开一个窗口

watcher 窗口：用于检测编译进度与错误

### IDEA 原生编译

[Remold](https://github.com/I-hope1/Remold) 推荐方式：把 IDEA 的 *Build and run using* 改成 `IntelliJ IDEA`，改完 **Ctrl+Shift+F9**只编译当前类

**注意输出目录**：IDEA 默认编译到 `out/production/...`，而 agent 监控的是 `build/classes/java/main`，两者需要统一目录

---

## 常见错误

### `Structural HotSwap Supported: false`

日志里同时出现 `JVM identified as Eclipse Adoptium...`，说明没换 JBR

原因通常有三：

1. `remold.jbr` 没配 / 路径错了；
2. 配了环境变量 `JBR_HOME` 但守护进程没重新启动；
3. JBR 路径存在但缺少 `bin/javaw.exe` ，即结构不完整；

修完后确认 `gradlew remoldInfo` 里打印的 `jbr java` 是指定的路径。

### `[HOTSWAP-SKIPPED] ... NO-ENHANCED-REDEFINE`

JBR 的增强重定义没打开，检查：

- `-XX:+AllowEnhancedClassRedefinition` 是否存在。
- `--add-exports=java.base/jdk.internal.misc=ALL-UNNAMED` 等三条 `--add-exports` 是否存在。

**重要**：JBR 的 `jdk.internal.misc` 反射被模块系统挡住时，增强重定义会静默不可用，表现就是“方法体热更正常，增加字段则跳过”。

### JBR 启动崩溃

现象：`EarlyWindowManager` 之后 `atio6axx.dll` 崩溃或 `SIGSEGV`，但日志里 `Structural HotSwap Supported: true`

说明 agent 本身没问题，有问题的是 JBR 的 OpenGL 后端

排查顺序：

1. **不带 agent**，用同一个 JBR 跑一次 `runClient`。依旧崩溃 → JBR 和显卡驱动的问题，和 Remold 无关；
2. 试 `-Dfml.earlyprogresswindow=false` 关掉早期窗口；
3. 换更新的 JBR 版本 / 更新显卡驱动 / 换 NVIDIA 环境；
4. 最后：`remold.jbr.scope=server`，client 不换 JVM，服务端照常热更。

### 启动时间过长

症状：卡在 `:writeClientLegacyClasspath` / `Resolve dependencies of :runtimeClasspath`

排查：

```bash
jcmd <daemonPid> Thread.print > threads.txt
```

看到类似：

```
Execution worker Thread 11
  ... sun.nio.ch.Net.poll
  ... NioSocketImpl.timedFinishConnect
  ... Gradle HttpClientHelper.performHead
  ... HttpResourceAccessor.getMetaData
```

说明 Gradle 在逐个对依赖仓库做 HEAD 元数据校验，而 `maven.neoforged.net` 在本机 443 端口连不通

实测数据（`gradlew writeClientLegacyClasspath`）：

| 场景 | 耗时 |
| --- | --- |
| 默认 | **333.5 s** |
| 把 HTTP 超时缩到 3s，冷守护进程 | 33.2 s |
| 同上，热守护进程 | 15.1 s |
| `--offline` | 2.3 ~ 3 s |

修复：

```properties
systemProp.org.gradle.internal.http.connectionTimeout=3000
systemProp.org.gradle.internal.http.socketTimeout=8000
```

或使用 `--offline` 离线启动，加依赖 / 升 NeoForge 时再用联网模式。

### 环境/工具链常见问题

- **日志里搜不到 `[Remold]`**：agent 的 banner 走 stdout，不进 log4j 文件 appender。看 Gradle 控制台，或者把输出重定向到文件。
- **用 PowerShell 写 `.bat` 时中文变 `?`**：`Set-Content -Encoding ASCII` 会把非 ASCII 字符替换掉。要么用英文注释，要么 `[IO.File]::WriteAllText` + `[Text.UTF8Encoding]`（但 cmd 对 UTF-8 BOM 也不友好，最好别写中文）。
- **给 javac/Groovy 读的文件带 BOM**：报 `错误: 非法字符: '\ufeff'`。写文件用 `[Text.UTF8Encoding]::new($false)`。
- **Gradle 守护进程缓存环境变量**：改 `JBR_HOME`、`JAVA_HOME` 之后记得 `gradlew --stop`。

---

## 设计细节

- **`file_shake_ms` 是防抖**：IDEA 一次保存可能写出好几个 class，等 1.2 秒再统一处理，避免半成品状态被加载。
- **新字段为什么可能不补初值**：Remold 只补零值；如果检测到对象上该字段已经有非默认值，会保守跳过，并打印提示。若需强制覆盖用 `@RemoldReinit(OVERWRITE)`（需要引入 `api` 依赖）。
- **匿名类不按类名对齐**：是按结构指纹；所以重排代码、改 lambda 位置不会串
- **重载保护**：删字段/改类型且堆里还有实例时，将放弃这次重定义，游戏将继续跑旧代码

---

## 参考

- Remold：<https://github.com/I-hope1/Remold>
- ModDevGradle 的 run 参数文件：`build/moddev/clientRunVmArgs.txt` / `clientRunProgramArgs.txt`
- Gradle 持续构建：`gradlew <task> -t`
- JBR 增强重定义：`-XX:+AllowEnhancedClassRedefinition` + `--add-exports` 三条
