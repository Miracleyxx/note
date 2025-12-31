------

## 🧩 一、核心问题定位：焦点被“系统拒绝”传递

在你当前的三层结构下：

| 层级    | 实体             | 进程       | 是否 Qt 控制                | 焦点行为                |
| ------- | ---------------- | ---------- | --------------------------- | ----------------------- |
| 第 1 层 | **WPS 进程**     | ✅ 独立进程 | ❌ 非 Qt 控制                | 只能接收 X11 的原生焦点 |
| 第 2 层 | **小容器 (Qt5)** | ✅ 独立进程 | ✅ Qt 控制，但非激活窗口     | 不接收 FocusInEvent     |
| 第 3 层 | **大容器 (Qt6)** | ✅ 主进程   | ✅ Qt 控制，但外部窗口未聚焦 | 不接收 FocusInEvent     |

------

### 🔍 真正的原因（X11 层面）

当你使用 `XReparentWindow()` 或 `createWindowContainer(QWindow::fromWinId(...))`
 把一个外部窗口（不管是小容器还是 WPS）嵌入另一个进程时：

- X11 把那个子窗口的父窗口改成了新的父窗口（例如 Qt6 的 container），
   但 **焦点不会跨进程自动继承**。
- 在此时：
  - 大容器不是最顶层激活窗口；
  - 小容器是外部（foreign）窗口，Qt 不会接收焦点；
  - X11 不允许一个客户端（Qt6）把焦点强制分配给另一个客户端（Qt5）除非显式调用 `XSetInputFocus`。

> 所以现在你看到的现象是：“鼠标移上去可以输入，移出就没法输入”，
>  因为一旦鼠标触发 EnterNotify/LeaveNotify，X11 的输入焦点会被回收或失效。

------

## 🧠 二、验证：用 `xprop` 或 `xwininfo` 看焦点状态

你可以在 Linux 下执行：

```bash
xprop -id <wps_window_id> | grep FOCUS
```

或：

```bash
xwininfo -root -tree | grep -i focus
```

你会看到焦点其实是停在 Qt6 的顶层窗口或 RootWindow 上，而不是小容器或 WPS 窗口上。
 这也就是你说的“两个容器都没有 focusIn”的根本原因。

------

## ⚙️ 三、真正有效的解决路径

现在我们要让系统认可：

> “当我鼠标移入我的 WPS 区域时，我这个区域内的窗口可以输入（即拥有输入焦点）。”

这有三条路线，分别适用于不同环境：

------

### ✅ 方案 A（推荐）：**在 Qt6 主进程中直接用 XSetInputFocus 驱动整个焦点逻辑**

在大容器（Qt6）中，定时或事件驱动调用：

```cpp
#include <X11/Xlib.h>
#include <QX11Info>

void WpsContainer::forceFocusToWps(::Window wpsWin) {
    Display *dpy = QX11Info::display();
    if (!dpy) return;
    XSetInputFocus(dpy, wpsWin, RevertToParent, CurrentTime);
    XFlush(dpy);
}
```

再结合一个全局鼠标事件过滤器：

```cpp
bool WpsContainer::eventFilter(QObject *obj, QEvent *ev) {
    if (ev->type() == QEvent::Enter) {
        forceFocusToWps(m_wpsWid);
    } else if (ev->type() == QEvent::Leave) {
        // 可选：暂不回收焦点，避免输入中断
    }
    return QWidget::eventFilter(obj, ev);
}
```

这相当于**直接告诉 X11**：
 “我的焦点交给这个窗口，不管 Qt 认不认。”

✅ 优点：

- 无需依赖小容器（Qt5）进程。
- 你已经验证过它在定时器场景下有效。
- 可稳定维持 WPS 输入。

⚠️ 缺点：

- 这种方式是“强制式焦点管理”，点击其他窗口后要重新激活。

------

### ✅ 方案 B：让小容器进程注册“焦点代理”Socket

既然 Qt6 → Qt5 焦点传递不通，我们就绕开 Qt：

在小容器（Qt5）中：

```cpp
connect(qApp, &QGuiApplication::focusWindowChanged, this, [this](QWindow *w){
    if (w) sendFocusChangeToQt6(true);
    else sendFocusChangeToQt6(false);
});
```

在大容器（Qt6）收到消息后执行：

```cpp
if (hasFocus)
    forceFocusToWps(m_wpsWid);
```

✅ 优点：

- 双向可控。
- 可扩展更多状态（例如点击、失焦、输入法状态）。

------

### ✅ 方案 C：系统级 Hook（XSelectInput）

你可以在 Qt6 的大容器里手动监听小容器或 WPS 窗口的焦点事件：

```cpp
Display *dpy = QX11Info::display();
XSelectInput(dpy, m_wpsWid, FocusChangeMask);
```

然后在 nativeEventFilter 中捕获 `FocusIn / FocusOut` 事件：

```cpp
if (ev->type == FocusIn && ev->xfocus.window == m_wpsWid) {
    qDebug() << "WPS got focus";
}
```

这能检测到焦点状态是否真正被系统授予。

------

## 💡 四、跨平台怎么办？

遗憾的是：

- 这种焦点机制是 **X11 独有**；
- 在 Wayland、Windows、macOS 上都没有 API 能跨进程直接设置输入焦点；
- 所以 Qt 官方不会提供“统一跨平台”的解决方法。

👉 你的场景（嵌入第三方独立进程窗口）本身属于 *“平台特定的 hack”*，
 因此“完美跨平台”在逻辑上不存在。

不过：

> 你可以封装 `setNativeInputFocusToWps()` 为统一接口，内部针对平台分别实现：
>
> - Linux: XSetInputFocus
> - Windows: SetFocus
> - macOS: 不支持，忽略

------

## ✅ 五、推荐稳定策略（真实可落地）

| 功能                                 | 实现方式                                                    |
| ------------------------------------ | ----------------------------------------------------------- |
| 鼠标进入区域时获取输入焦点           | 大容器安装事件过滤器，调用 `setNativeInputFocusToWps(true)` |
| 鼠标移出区域时保持输入（不回收焦点） | 不处理 `Leave`                                              |
| 点击其他窗口后恢复焦点               | 监听 `QApplication::applicationStateChanged()` 重新设置焦点 |
| Wayland 场景下                       | 降级：不做处理，仅依赖系统输入                              |

------

## 🧭 总结一句话

> **Qt 自身无法跨进程传递 focus 事件；必须在 X11 层主动用 XSetInputFocus 驱动。**
>  “小的和大的都收不到 focus”是因为它们都只是窗口管理的壳，没有真正持有系统焦点。

------

如果你愿意，我可以帮你写出一个：

> 🧩 **WpsFocusController 类**
>  自动监听 Qt 焦点状态、X11 鼠标移动、外部激活状态，确保你嵌入的 WPS 始终可输入。

是否希望我帮你写出这份可直接集成的类代码？

非常好，这个问题问得非常关键——因为你现在已经搞清楚**运行时**没问题，接下来要明确**编译时依赖怎么写**，否则跨平台编译会有坑。
 我们来完整地分层说明👇

------

## 🧱 一、你现在的场景概况

- 工程是 **Qt 5.15.1**；
- 用到了 **`XSetInputFocus()`、`XOpenDisplay()`** 等函数；
- 在 **Linux 下**运行，桌面可能是 X11 或 Wayland；
- 你希望程序能正常编译、链接，并且：
  - ✅ 在 X11 下能正常工作；
  - ⚙️ 在 Wayland 下至少能启动（X11 调用无效也没关系）。

------

## ⚙️ 二、正确的编译链接方式

### ✅ 如果你用 **qmake**：

在 `.pro` 文件里加入：

```pro
unix {
    QT += widgets gui
    CONFIG += c++11

    # X11 依赖
    LIBS += -lX11
    INCLUDEPATH += /usr/include
}
```

> 💡 Qt 自带的 headers 不会包含 X11 的定义，
>  因为那是纯系统级 API，必须自己链接 `libX11.so`。

------

### ✅ 如果你用 **CMake**：

```cmake
find_package(Qt5 REQUIRED COMPONENTS Widgets Gui)

# 查找 X11 库（系统会自动提供）
find_package(X11 REQUIRED)

add_executable(myapp main.cpp wps_container.cpp ...)
target_link_libraries(myapp
    PRIVATE
        Qt5::Widgets
        Qt5::Gui
        ${X11_LIBRARIES}
)
target_include_directories(myapp PRIVATE ${X11_INCLUDE_DIR})
```

> 📘 `find_package(X11)` 是跨平台友好的写法：
>
> - 在 X11 平台上会找到 `/usr/lib/libX11.so`；
> - 在 macOS / Windows 上自动跳过（不会编译 X11 部分）；
> - 你也可以在源码中用 `#ifdef Q_OS_UNIX` 或 `#ifdef Q_OS_LINUX` 来保护 X11 代码。

------

## 🔒 三、代码层面的条件保护（必须加）

```cpp
#ifdef Q_OS_UNIX
#include <X11/Xlib.h>
#endif
```

然后使用时：

```cpp
#ifdef Q_OS_UNIX
Display* dpy = XOpenDisplay(nullptr);
if (dpy) {
    XSetInputFocus(dpy, targetWin, RevertToParent, CurrentTime);
    XCloseDisplay(dpy);
} else {
    qWarning("X11 not available (likely running under Wayland)");
}
#endif
```

```c++
#include <QWidget>
#include <QDebug>

#ifdef Q_OS_UNIX
#include <QX11Info>
#include <X11/Xlib.h>
#endif

void activateWindowCrossPlatform(QWidget* target)
{
    if (!target)
        return;

#ifdef Q_OS_UNIX
    // 优先：如果是 X11，就用 X11 原生方式
    if (QX11Info::isPlatformX11()) {
        Display* dpy = QX11Info::display();
        if (dpy) {
            WId winId = target->winId();
            XSetInputFocus(dpy, winId, RevertToParent, CurrentTime);
            XFlush(dpy);
            qDebug() << "[Focus] X11 focus set via XSetInputFocus()";
            return;
        }
    }
#endif

    // 通用 Qt 方式（Wayland、Windows、macOS）
    if (target->isMinimized())
        target->showNormal();

    target->raise();                                  // 提升层级
    target->activateWindow();                         // 请求系统激活
    target->setFocus(Qt::ActiveWindowFocusReason);    // 内部聚焦
}

```

```C++
#include <QApplication>
#include <QWidget>
#include <QDebug>

#ifdef Q_OS_UNIX
#include <QX11Info>
#include <QLibrary>
#include <X11/Xlib.h>
#endif

/**
 * @brief X11 焦点辅助类（单例）
 *
 * (已更新) 封装了对 libX11 的动态加载和 XSetInputFocus 的解析。
 * 核心方法接受原生 WId (Window ID)。
 */
class X11FocusHelper {
public:
    // 定义 XSetInputFocus 的函数指针类型
    using XSetInputFocusFunc = int(*)(Display*, Window, int, Time);

    /**
     * @brief 获取 X11FocusHelper 的全局唯一实例
     */
    static X11FocusHelper& instance() {
        static X11FocusHelper inst;
        return inst;
    }

    /**
     * @brief 检查 X11 特定的聚焦功能是否可用
     */
    bool available() const { return _available; }

    /**
     * @brief (核心方法) 尝试使用 XSetInputFocus 设置焦点 (接受原生 WId)
     * @param windowId 目标的 X11 Window ID (在 Qt 中 typedef 为 WId)
     * @return true 如果调用成功
     */
    bool setFocus(WId windowId) {
        if (!_available || windowId == 0) // 0 通常是无效句柄
            return false;

        Display* dpy = QX11Info::display();
        if (!dpy) {
            qDebug() << "[X11] Failed to get X11 Display.";
            return false;
        }

        // 直接调用缓存的函数指针
        _setInputFocus(dpy, windowId, RevertToParent, CurrentTime);
        // 注意：XSetInputFocus 本身是 int 返回值，但通常异步执行。
        // 这里我们假设调用即成功，除非 dpy 为空。
        return true;
    }

    /**
     * @brief (便捷重载) 尝试使用 XSetInputFocus 设置焦点 (接受 QWidget)
     * @param target 目标窗口 QWidget
     * @return true 如果调用成功
     */
    bool setFocus(QWidget* target) {
        if (!target)
            return false;
        
        return setFocus(target->winId()); // 委托给 WId 版本
    }

    // 删除拷贝构造和赋值操作
    X11FocusHelper(const X11FocusHelper&) = delete;
    X11FocusHelper& operator=(const X11FocusHelper&) = delete;

private:
    /**
     * @brief 私有构造函数，在 instance() 首次调用时执行
     */
    X11FocusHelper() {
#ifdef Q_OS_UNIX
        if (!QX11Info::isPlatformX11()) {
            qDebug() << "[X11] Not X11 platform, skip load.";
            _available = false;
            return;
        }

        _lib.setFileName("libX11.so.6");
        if (!_lib.load()) {
            qDebug() << "[X11] libX11.so.6 not found, skip.";
            _available = false;
            return;
        }

        _setInputFocus = reinterpret_cast<XSetInputFocusFunc>(
            _lib.resolve("XSetInputFocus")
        );

        if (_setInputFocus) {
            qDebug() << "[X11] XSetInputFocus loaded successfully.";
            _available = true;
        } else {
            qDebug() << "[X11] Failed to resolve XSetInputFocus.";
            _available = false;
            _lib.unload();
        }
#else
        _available = false;
#endif
    }

    /**
     * @brief 析构函数，在程序退出时释放库资源
     */
    ~X11FocusHelper() {
        if (_lib.isLoaded())
            _lib.unload();
    }

private:
    QLibrary _lib;
    XSetInputFocusFunc _setInputFocus = nullptr;
    bool _available = false;
};


// -----------------------------------------------------------------
//                          使用示例
// -----------------------------------------------------------------

/**
 * @brief (用法 1) 跨平台的 QWidget 激活函数 (带回退)
 *
 * 适用于目标是 QWidget，且需要 Wayland/Windows/macOS 兼容性的情况。
 */
void activateWindowCrossPlatform(QWidget* target)
{
    if (!target)
        return;

#ifdef Q_OS_UNIX
    auto& x11Helper = X11FocusHelper::instance();
    if (x11Helper.available()) {
        // 使用 QWidget 重载
        if (x11Helper.setFocus(target)) { 
            qDebug() << "[X11] Focus set via cached function (QWidget).";
            return; 
        }
    }
#endif

    // ---- 通用回退 (Fallback) ----
    qDebug() << "[Qt] Using fallback focus/activation.";
    if (target->isMinimized())
        target->showNormal();
    
    target->raise();
    target->activateWindow();
    target->setFocus(Qt::ActiveWindowFocusReason);
}

/**
 * @brief (用法 2) 仅 X11 的原生句柄聚焦函数 (无回退)
 *
 * 适用于目标不是 QWidget，而是已知的 WId 的情况。
 * (注意：此函数在非 X11 平台无效)
 */
void focusNativeWindow(WId nativeHandle)
{
    if (nativeHandle == 0)
        return;

#ifdef Q_OS_UNIX
    auto& x11Helper = X11FocusHelper::instance();

    // 重要：对于非 Qt 句柄，我们没有 Qt API 可以回退。
    // 只能尝试 X11 方法。
    if (x11Helper.available()) {
        // 使用 WId 核心方法
        if (x11Helper.setFocus(nativeHandle)) { 
            qDebug() << "[X11] Focus set via cached function (Native WId).";
        } else {
            qDebug() << "[X11] Failed to set focus to native WId.";
        }
    } else {
        qDebug() << "[X11] Helper not available. Cannot focus native handle.";
    }
#else
    // 在 Windows/macOS 上，这个函数无法操作 X11 WId
    qDebug() << "Cannot focus native X11 WId on non-Unix platform.";
#endif
}
```

```cmake
QT += core gui widgets

# 引入 x11extras 以使用 QX11Info
QT += x11extras

# (可选但推荐) 定义一个宏，用于条件编译
unix:!macx {
    DEFINES += USE_X11_OPTIONAL
}
```



✅ **这样做的结果：**

- 编译时只在 Linux 下启用；
- Wayland 环境仍然能运行；
- macOS / Windows 不受影响；
- 链接时不会报错；
- 运行时不会崩溃。

------

## 🧩 四、总结表格

| 平台                               | 编译方式                       | 是否需要 `-lX11` | 是否能运行 | 焦点逻辑 |
| ---------------------------------- | ------------------------------ | ---------------- | ---------- | -------- |
| **X11 桌面 (Ubuntu, Deepin, KDE)** | ✅ 正常编译                     | ✅ 必须           | ✅ 正常     | ✅ 生效   |
| **Wayland 桌面 (GNOME 46)**        | ✅ 正常编译                     | ✅ 必须           | ✅ 可运行   | ⚠️ 不生效 |
| **Windows / macOS**                | ✅ 正常编译                     | ❌ 不链接         | ✅ 可运行   | ❌ 无效   |
| **CI/CD 构建 (Ubuntu)**            | ✅ 推荐使用 `find_package(X11)` | ✅                | ✅          | ✅        |

------

## ✨ 五、推荐做法总结（一句话版本）

> 💡 **结论：**
>
> - **编译时链接 `libX11.so`（`-lX11` 或 `find_package(X11)`）是正确且安全的做法；**
> - **Qt 自身不提供这类焦点操作 API；**
> - **通过宏保护后可跨平台编译，Wayland 下自动失效但可启动；**
> - **不建议改成 Qt 内部焦点机制，因为它无法控制嵌入的 WPS 外进程窗口。**

------
