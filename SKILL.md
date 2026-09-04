---
name: upgrade-libxposed-102
description: Comprehensive guide and standard procedure for upgrading Android Xposed modules from legacy XposedBridge API (54-93) to modern LibXposed API 102 (LSPosed 1.9.3+ / Modern Xposed Module API). Use whenever the user asks to upgrade an Xposed module to API 102 or Modern Xposed API.
---

# Xposed 模块升级至 LibXposed API 102 完全指南

本文档总结了将传统 Xposed 模块（基于 XposedBridge API 54-93）升级至现代 **LibXposed API 102**（适用于 LSPosed 1.9.3+、Vector 等现代框架）的标准流程、代码范式、元数据规范与性能优化策略。

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
| **Zygote 注入** | 支持 `initZygote` 全局注入 | **禁止 Zygote 注入**，仅注入到作用域目标进程 |
| **混淆/R8 支持** | 需严格保留 Hook 方法名与类名 | 原生支持 R8 混淆，声明适配规则即可 |

---

## 二、标准迁移四步法

### 第一步：构建与依赖迁移 (`build.gradle`)

1. **更新依赖项**：
   在模块的 `build.gradle`（如 `app/build.gradle` 或 `hook/build.gradle`）中替换依赖：
   ```groovy
   dependencies {
       // 移除: compileOnly 'de.robv.android.xposed:api:82'
       // 引入现代 LibXposed 102 API
       compileOnly 'io.github.libxposed:api:102.0.0'
       compileOnly 'androidx.annotation:annotation:1.5.0'
   }
   ```

2. **Java 编译兼容性**：
   ```groovy
   android {
       compileOptions {
           sourceCompatibility JavaVersion.VERSION_11
           targetCompatibility JavaVersion.VERSION_11
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
com.example.myhook.MainHook
```

#### 3. `scope.list`（推荐作用域）
每行一个目标包名，声明模块默认需要注入的应用：
```text
com.android.chrome
com.microsoft.emmx
com.example.targetapp
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

#### 1. 基础模板与生命周期

```java
package com.example.myhook;

import android.util.Log;
import androidx.annotation.NonNull;
import io.github.libxposed.api.XposedInterface;
import io.github.libxposed.api.XposedModule;

public class MainHook extends XposedModule {

    private static final String TAG = "MyHookModule";
    private volatile boolean mHooked = false;

    public MainHook() {
        super();
    }

    /**
     * 宿主应用 ClassLoader 就绪且即将创建 Application 时触发
     */
    @Override
    public void onPackageReady(@NonNull PackageReadyParam param) {
        if (mHooked) return;
        synchronized (this) {
            if (mHooked) return;
            mHooked = true;
            initHooks(param.getPackageName(), param.getClassLoader());
        }
    }

    /**
     * 热重载准备回调（运行在旧版本代码中）
     * 返回 true 允许框架替换为新版本
     */
    @Override
    public boolean onHotReloading(@NonNull HotReloadingParam param) {
        return true;
    }

    /**
     * 热重载完成回调（运行在新版本代码中）
     */
    @Override
    public void onHotReloaded(@NonNull HotReloadedParam param) {
        // 解除旧版本注册的所有 Hook
        param.getOldHookHandles().forEach(XposedInterface.HookHandle::unhook);
        synchronized (this) {
            mHooked = false;
            initHooks("hot-reload", getClass().getClassLoader());
        }
    }

    private void initHooks(String packageName, ClassLoader classLoader) {
        // 执行具体 Hook 逻辑
    }
}
```

#### 2. 核心 Hook 语法对照

| 目标操作 | 传统 Xposed 写法 | LibXposed 102 现代写法 |
| :--- | :--- | :--- |
| **查找并 Hook 方法** | `XposedHelpers.findAndHookMethod("类名", cl, "方法", argTypes..., callback)` | `Method m = Clazz.class.getDeclaredMethod("方法", argTypes...);`<br>`hook(m).intercept(chain -> { ... });` |
| **执行原方法** | 默认直接执行；调用 `param.getResult()` | `Object result = chain.proceed();` |
| **修改参数并调用** | `param.args[0] = newVal;` | `Object result = chain.proceed(new Object[]{ newVal, ... });` |
| **拦截并替换返回值** | `param.setResult(customVal);` | **直接返回自定义值**（不调用 `chain.proceed()`）：<br>`hook(m).intercept(chain -> customVal);` |
| **获取入参** | `param.args[i]` | `chain.getArg(i)` 或 `chain.getArgs()` |
| **获取 this 实例** | `param.thisObject` | `chain.getThisObject()` |
| **Hook 构造函数** | `findAndHookConstructor(...)` | `Constructor<?> ctor = ...;`<br>`hook(ctor).intercept(chain -> ...);` |
| **Hook 静态代码块** | 传统不支持标准 API | `hookClassInitializer(TargetClass.class).intercept(...)` |

---

## 三、高频 Hook 场景与性能深度优化模式

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

## 四、升级常见陷阱与排查清单

1. **`assets/xposed_init` 是否仍需保留？**
   - 现代 API 102 仅读取 `META-INF/xposed/java_init.list`。如果保留 `assets/xposed_init`，旧版框架可能尝试使用传统反射加载它而报错，推荐直接移除或保持为空。
2. **`compileSdk` 与 Android SDK 平台**：
   - 建议将 `compileSdk` 与 `targetSdk` 升级至 Android 33 或 34，以便顺利使用现代接口和 Java 11/17 语言特性。
3. **Gradle 兼容性**：
   - 如果构建环境使用的是 JDK 17 / 21，必须确保 Gradle Wrapper 升级至 `7.4+`（推荐 `7.5.1` 或 `8.x`），AGP 升级至 `7.3+`。
4. **验证模块识别**：
   - 使用解压工具查看最终 APK 根目录，必须包含：
     - `META-INF/xposed/module.prop`
     - `META-INF/xposed/java_init.list`
     - `META-INF/xposed/scope.list`
