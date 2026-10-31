---
name: upgrade-libxposed-102
description: Comprehensive guide and standard procedure for upgrading Android Xposed modules from legacy XposedBridge API (54-93) to modern LibXposed API 102 (LSPosed 1.9.3+ / Modern Xposed Module API). Use whenever the user asks to upgrade an Xposed module to API 102 or Modern Xposed API.
---

# Xposed 模块升级至 LibXposed API 102 完全指南

本文档总结了将传统 Xposed 模块（基于 XposedBridge API 54-93）升级至现代 **LibXposed API 102**（适用于 LSPosed 1.9.3+、Vector 等现代框架）的标准流程、代码范式、系统服务挂钩、跨进程通信、元数据规范与性能优化策略。

后续对任何其他模块执行升级指示时，可直接参照本规范执行。

---

## 一、核心变更对比

| 特性 / 维度 | 传统 Xposed (API 54-93) | 现代 LibXposed (API 102) |
| :--- | :--- | :--- |
| **依赖坐标** | `de.robv.android.xposed:api:82` (或 93) | `io.github.libxposed:api:102.0.0` |
| **入口基类/接口** | `IXposedHookLoadPackage`, `IXposedHookZygoteInit` | 继承 `io.github.libxposed.api.XposedModule` |
| **入口声明** | `assets/xposed_init` | `META-INF/xposed/java_init.list` |
| **模块属性配置** | `AndroidManifest.xml` 中的 `<meta-data>` | `META-INF/xposed/module.prop` |
| **作用域声明** | `xposedscope` resource array | `META-INF/xposed/scope.list` |
| **Hook 编程模型** | `XC_MethodHook` (`before` / `after`) 回调 | 类 OkHttp 的链式拦截器 `Hooker` (`chain.proceed()`) |
| **热重载 (Hot Reload)**| 不支持（需重启目标应用甚至系统） | 原生支持（重写 `onHotReloading` / `onHotReloaded`） |
| **系统服务/Zygote** | `initZygote` 全局注入 | 专用 `onSystemServerStarting`；严格禁用全局 Zygote 注入 |
| **跨进程配置同步** | 传统 `XSharedPreferences`（Android 7+ 及 SELinux 严格环境失效） | 推荐 `Settings.Global` + `ContentObserver` 实时热同步（零常驻） |
| **特权安全防护** | 隐式跟随调用方身份，高版本易报 Permission Denial | 显式使用 `Binder.clearCallingIdentity()` 恢复系统最高特权 |
| **混淆/R8 支持** | 需严格保留 Hook 方法名与类名 | 原生支持 R8 混淆，声明适配规则即可 |

---

## 二、标准迁移四步法

### 第一步：构建与依赖迁移 (`build.gradle`)

1. **更新依赖项**：
   在模块的 `build.gradle`（如 `app/build.gradle`）中替换依赖：
   ```groovy
   dependencies {
       // 移除: compileOnly 'de.robv.android.xposed:api:82'
       // 引入现代 LibXposed 102 API
       compileOnly 'io.github.libxposed:api:102.0.0'
       compileOnly 'androidx.annotation:annotation:1.5.0'
   }
   ```

2. **Java / Kotlin 编译兼容性**：
   ```groovy
   android {
       compileOptions {
           sourceCompatibility JavaVersion.VERSION_17
           targetCompatibility JavaVersion.VERSION_17
       }
       kotlinOptions {
           jvmTarget = "17"
       }
   }
   ```

3. **配置 ProGuard / R8 规则 (`proguard-rules.pro`)**：
   ```proguard
   -dontwarn io.github.libxposed.annotation.**
   -adaptresourcefilecontents META-INF/xposed/java_init.list
   -keep,allowoptimization,allowobfuscation public class * extends io.github.libxposed.api.XposedModule {
       public <init>();
   }
   ```

---

### 第二步：现代元数据文件配置 (`META-INF/xposed/`)

在模块源码目录 `src/main/resources/META-INF/xposed/` 下创建以下文件：

#### 1. `module.prop`（模块元数据）
```properties
minApiVersion=102
targetApiVersion=102
autoHotReload=true
exceptionMode=protective
```
- `minApiVersion`: 最低支持的 API 版本，设为 `102`。
- `targetApiVersion`: 目标 API 版本，设为 `102`。
- `autoHotReload`: 设为 `true` 表示模块更新安装时自动触发热重载。
- `exceptionMode`: `protective`（默认保护模式，Hook 抛异常不影响宿主崩溃）或 `passthrough`。

#### 2. `java_init.list`（入口类声明）
单行声明模块入口全限定类名：
```text
io.github.marukon.imeauto.hook.ImeHookEntry
```

#### 3. `scope.list`（推荐作用域）
每行一个目标包名，声明模块默认需要注入的应用。若需要挂钩系统核心服务，必须包含 `android`：
```text
android
com.android.chrome
com.microsoft.emmx
```

---

### 第三步：清单文件配置 (`AndroidManifest.xml`)

更新模块的 `AndroidManifest.xml`：
```xml
<application ...>
    <!-- 传统元数据保持兼容，关键将 xposedminversion 设为 102 -->
    <meta-data
        android:name="xposedmodule"
        android:value="true" />
    <meta-data
        android:name="xposeddescription"
        android:value="模块功能描述" />
    <meta-data
        android:name="xposedminversion"
        android:value="102" />
    <meta-data
        android:name="xposedscope"
        android:resource="@array/xposed_scope" />
</application>
```

---

### 第四步：Hook 逻辑重构 (`XposedModule`)

#### 1. 基础入口结构（Kotlin 现代范式）

```kotlin
package io.github.marukon.imeauto.hook

import io.github.libxposed.api.XposedInterface
import io.github.libxposed.api.XposedModule
import io.github.libxposed.api.XposedModuleInterface.ModuleLoadedParam
import io.github.libxposed.api.XposedModuleInterface.PackageReadyParam
import io.github.libxposed.api.XposedModuleInterface.SystemServerStartingParam

class MainHook : XposedModule() {

    override fun onModuleLoaded(param: ModuleLoadedParam) {
        super.onModuleLoaded(param)
        // 进程初始化，支持获取 param.processName 与 param.isSystemServer
    }

    /**
     * system_server 专属初始化回调
     */
    override fun onSystemServerStarting(param: SystemServerStartingParam) {
        super.onSystemServerStarting(param)
        SystemHook.init(this, param.classLoader)
    }

    /**
     * 目标应用或系统包 ClassLoader 就绪时触发
     */
    override fun onPackageReady(param: PackageReadyParam) {
        super.onPackageReady(param)

        // 挂钩系统框架层
        if (param.packageName == "android" || param.packageName == "system") {
            SystemHook.init(this, param.classLoader)
            return
        }

        // 模块自身激活自检测（Self-Hooking）
        if (param.packageName == "io.github.marukon.imeauto") {
            try {
                val clazz = Class.forName("io.github.marukon.imeauto.data.ConfigManager", false, param.classLoader)
                val method = clazz.getDeclaredMethod("isModuleActive")
                hook(method)
                    .setPriority(XposedInterface.PRIORITY_HIGHEST)
                    .intercept { true }
            } catch (t: Throwable) {
                log(android.util.Log.ERROR, "MainHook", t)
            }
        }
    }
}
```

#### 2. 核心 Hook 语法对照

| 目标操作 | 传统 Xposed 写法 | LibXposed 102 现代写法 |
| :--- | :--- | :--- |
| **查找并 Hook 方法** | `XposedHelpers.findAndHookMethod(...)` | `val m = Clazz::class.java.getDeclaredMethod(...)`<br>`hook(m).setPriority(...).intercept { chain -> ... }` |
| **执行原方法** | 默认直接执行；调用 `param.getResult()` | `val result = chain.proceed()` |
| **修改参数并调用** | `param.args[0] = newVal` | `val result = chain.proceed(arrayOf(newVal, ...))` |
| **拦截并替换返回值** | `param.setResult(customVal)` | **直接返回自定义值**（不调用 `chain.proceed()`）：<br>`hook(m).intercept { customVal }` |
| **获取入参** | `param.args[i]` | `chain.getArg(i)` 或 `chain.args` |
| **获取 this 实例** | `param.thisObject` | `chain.thisObject` |
| **Hook 构造函数** | `findAndHookConstructor(...)` | `val ctor = ...`<br>`hook(ctor).intercept { chain -> ... }` |
| **设置执行优先级** | `XposedBridge.hookMethod(..., priority)` | `hook(m).setPriority(XposedInterface.PRIORITY_HIGHEST)` |

---

## 三、系统服务级 Hook（SystemServer Hooking）与特权防御

### 1. 系统 Context 安全获取与延迟初始化
在 `system_server` 中，无法直接使用普通 `ApplicationContext`。通过双重保底机制安全获取系统 Context：
```kotlin
object HookUtils {
    fun getSystemContext(): Context? {
        // 保底策略 1: AppGlobals.getInitialApplication()
        try {
            val appGlobalsClass = Class.forName("android.app.AppGlobals")
            val method = appGlobalsClass.getDeclaredMethod("getInitialApplication")
            val app = method.invoke(null) as? Context
            if (app != null) return app
        } catch (ignored: Throwable) {}

        // 保底策略 2: ActivityThread.currentActivityThread().getSystemContext()
        try {
            val activityThreadClass = Class.forName("android.app.ActivityThread")
            val currentActivityThreadMethod = activityThreadClass.getDeclaredMethod("currentActivityThread")
            val activityThread = currentActivityThreadMethod.invoke(null)
            if (activityThread != null) {
                val getSystemContextMethod = activityThreadClass.getDeclaredMethod("getSystemContext")
                val ctx = getSystemContextMethod.invoke(activityThread) as? Context
                if (ctx != null) return ctx
            }
        } catch (ignored: Throwable) {}
        return null
    }
}
```

### 2. 跨进程特权隔离与权限恢复（Binder Identity 防御）
**核心问题**：在 Android 14/15/16（如 ColorOS 16 / HyperOS / One UI）下，挂钩系统服务（如 AMS/ATMS/IMMS）的方法经常运行在处理客户端 IPC 的 Binder 线程中，当前线程 UID 属于调用方应用（如 UID 10234）。  
若此时直接调用系统敏感 API（如 `Settings.Secure.putStringForUser`），系统将抛出 `SecurityException: Permission Denial` 崩溃。

**标准解法**：必须使用特权提升与恢复模式：
```kotlin
val token = Binder.clearCallingIdentity()
try {
    // 恢复为 system_server (UID 1000) 特权身份执行敏感调用
    Settings.Secure.putStringForUser(resolver, key, value, userId)
} finally {
    Binder.restoreCallingIdentity(token)
}
```

### 3. 现代零常驻跨进程配置热同步（Settings.Global + ContentObserver）
针对 Android 7+ 及 SELinux 严格模式下传统 `XSharedPreferences` 失效的问题：
- **控制台 App 端**：将配置序列化为 JSON 字符串，直接写入 `Settings.Global`：
  ```kotlin
  Settings.Global.putString(context.contentResolver, "module_config_key", jsonString)
  ```
- **系统挂钩端 (`system_server`)**：注册 `ContentObserver` 监听配置变化，秒级热重载：
  ```kotlin
  val uri = Settings.Global.getUriFor("module_config_key")
  context.contentResolver.registerContentObserver(uri, false, object : ContentObserver(Handler(Looper.getMainLooper())) {
      override fun onChange(selfChange: Boolean) {
          reloadConfigFromGlobalSettings()
      }
  })
  ```
- **优势**：无需常驻后台 Service，无需无障碍辅助功能，开销为零，防杀后台，全 Android 版本兼容。

---

## 四、轻量级反射增强工具模板（替代废弃的 XposedHelpers）

由于 LibXposed 102 不再捆绑 `XposedHelpers`，且引入旧版 jar 会污染类路径，推荐在项目中维护独立的轻量反射工具类：

```kotlin
object ReflectionUtils {
    fun findFirstMethodNamed(clazz: Class<*>, name: String): Method? {
        var current: Class<*>? = clazz
        while (current != null && current != Any::class.java) {
            for (m in current.declaredMethods) {
                if (m.name == name) {
                    m.isAccessible = true
                    return m
                }
            }
            current = current.superclass
        }
        return null
    }

    fun findMethodExact(clazz: Class<*>, name: String, vararg paramTypes: Class<*>): Method {
        val m = clazz.getDeclaredMethod(name, *paramTypes)
        m.isAccessible = true
        return m
    }
}
```

---

## 五、高频 Hook 场景与性能深度优化模式

### 针对高频调用（如 `isEnabled`、`View` 绘制、触摸派发）的黄金优化法则：

1. **零开销快速路径（嗅探直通）**：
   若方法是状态查询方法（如 `isEnabled()`、`isLoaded()`），先调用轻量级的原生字段读取 `chain.proceed()`。如果真实状态本身即为假，直接返回假，跳过所有后续逻辑与堆栈检查。

2. **Android 14+ (API 34+) 采用 `StackWalker` 惰性遍历**：
   传统 `new Throwable().getStackTrace()` 会导致全线程栈深度展开（Native Unwind）和几百个字符串分配，极易引起 GC 掉帧。
   使用 Java 9 / Android 14+ 的 `java.lang.StackWalker`，按需逐层扫描，零全栈分配。

3. **Android < 34 动态栈定位（杜绝硬编码）**：
   仅单次执行 `new Throwable().getStackTrace()`，向下遍历查找目标方法所在的系统类（如 `AccessibilityManager`），紧随其后的栈帧即为真实调用者。绝不硬编码 `[3]` 或 `[4]`，保证跨 LSPosed 版本稳定运行。

#### 性能优化示范代码：
```java
// 检查 Android 14+ StackWalker
private static final boolean HAS_STACK_WALKER;
static {
    boolean has;
    try {
        Class.forName("java.lang.StackWalker");
        has = (Build.VERSION.SDK_INT >= 34);
    } catch (Throwable t) {
        has = false;
    }
    HAS_STACK_WALKER = has;
}

// 高频方法 Hook 示范
Method isEnabled = AccessibilityManager.class.getDeclaredMethod("isEnabled");
hook(isEnabled).intercept(chain -> {
    Object real = chain.proceed();
    if (Boolean.FALSE.equals(real)) {
        return false; // 系统本就是关的，零开销直通返回
    }
    return isCallerWhitelisted() ? real : false;
});

private static boolean isCallerWhitelisted() {
    if (HAS_STACK_WALKER) {
        try {
            return Api34StackWalker.check();
        } catch (Throwable ignored) {}
    }
    return checkLegacy();
}

@RequiresApi(Build.VERSION_CODES.UPSIDE_DOWN_CAKE)
private static final class Api34StackWalker {
    private static final StackWalker WALKER = StackWalker.getInstance(StackWalker.Option.RETAIN_CLASS_REFERENCE);

    static boolean check() {
        return Boolean.TRUE.equals(WALKER.walk(frames -> {
            boolean foundTarget = false;
            var it = frames.iterator();
            while (it.hasNext()) {
                var f = it.next();
                if ("android.view.accessibility.AccessibilityManager".equals(f.getClassName())) {
                    foundTarget = true;
                } else if (foundTarget) {
                    return isPackageAllowed(f.getClassName());
                }
            }
            return false;
        }));
    }
}
```

---

## 六、CI/CD 与 LSPosed 官方模块仓库集成规范

### 1. 官方仓库命名与发版契约
- 仓库组织：`https://github.com/Xposed-Modules-Repo/<package_name>`
- 包名与仓库名必须严格一致（例如 `io.github.marukon.imeauto`）。
- Release 标签推荐使用 `vX.Y.Z` 规范，附带已签名的 `app-release.apk`。

### 2. GitHub Actions 自动化关键防坑实践
- **跨仓库推送权限陷阱（403 Permission Denied）**：
  使用 Personal Access Token 向 `Xposed-Modules-Repo` 推送代码或创建 Release 时，`actions/checkout` 会默认持久化当前主仓库内置的 bot 凭证。必须设置 `persist-credentials: false`，并在推送步骤前移除本地的 `.extraheader`：
  ```yaml
  - name: Checkout Source
    uses: actions/checkout@v4
    with:
      persist-credentials: false

  - name: Push to Repo
    env:
      LSP_TOKEN: ${{ secrets.LSP_REPO_TOKEN }}
    run: |
      git config --local --unset-all http.https://github.com/.extraheader || true
      git remote add lsposed "https://${LSP_TOKEN}@github.com/Xposed-Modules-Repo/target-repo.git"
      git push lsposed main:main --force
  ```
- **文档修改免跑 CI**：
  在 `ci.yml` 中声明 `paths-ignore: ['**.md', 'art/**', 'docs/**', 'LICENSE', '.gitignore']`，修改 README 或文档资源时跳过昂贵的 Gradle 编译流程。
- **签名机密安全注入与清理**：
  将 JKS 证书 Base64 编码保存在 GitHub Secrets（`KEYSTORE_BASE64`），CI 构建前动态生成，构建完成后在 `always()` 阶段彻底清除，避免凭据泄漏。
