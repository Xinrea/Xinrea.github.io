---
title: "Qt OpenglWindow 异形窗口的实现"
draft: false
---

在学习熟悉 CubismSDK 的时候，曾给轴伊Joi制作过一个简单的 Live2D 桌面宠物；由于是在官方样例的基础上进行的修改，因此程序主题通过 glew + glfw 来进行实现。由于桌面宠物的特殊性（需要尽可能减少对桌面操作的影响），可以说是必须实现异形窗口。这个异形窗口与一般的需求还不太一样：通常异形窗口是静态的，仅以一张图片作为底图，有很多种方法可以实现，其中一种便是用蒙版（Mask）来实现，但这种方式在桌面宠物这种场景下显得有点尴尬。

对于用 OpenGL 动态渲染的桌面宠物来说，读取当前 Buffer 生成 Mask 是效率极低的。好在 Windows 下提供了 `SetLayeredWindowAttributes(hwnd,RGB(0, 0, 0), 255, LWA_COLORKEY)` 这一方法，直接进行键值抠图即可。但问题是其精度极低，渲染出的模型边缘会出现很明显的底色锯齿边缘。

在 glfw 中，通过 `glfwWindowHint(GLFW_TRANSPARENT_FRAMEBUFFER, GLFW_TRUE)` 可实现窗口 Buffer 新增 Alpha 通道；配合 `LWA_COLORKEY` 即可实现无锯齿边缘的 OpenGL 渲染的即时异形窗口。

然而最近在使用 Qt 对其重构的过程中，又遇到了异形窗口的这一问题。Qt 提供了两种使用 OpenGL 的方式，QOpenGLWindow 与 QOpenGLWidget；这两种方式在使用上几乎没有差异，可以很方便的互相转换。但在实现异形窗口的过程中遇到了问题。

通过实践发现，QOpenGLWidget 通过设置 `Qt::WA_TranslucentBackground` 可以很简单的实现透明背景；但 `LWA_COLORKEY` 只对 `WS_EX_LAYERED` 样式的窗口生效；经过测试，QOpenGLWidget 单独作为窗口时，无法设置 `WS_EX_LAYERED` 样式，因此无法实现异形窗口。

QOpenGLWindow 可以直接设置 `WS_EX_LAYERED` 样式，但无法设置透明背景（可使用 `setFormat` 添加 alpha 通道属性，但配合 `LWA_COLORKEY` 会显示异常）；因此需要自行实现 `GLFW_TRANSPARENT_FRAMEBUFFER` 的功能。

通过阅读 glfw 源码，该设置相关的代码如下所示：

```cpp
#define DWM_BB_ENABLE 0x00000001
#define DWM_BB_BLURREGION 0x00000002
typedef struct
{
    DWORD dwFlags;
    BOOL fEnable;
    HRGN hRgnBlur;
    BOOL fTransitionOnMaximized;
} DWM_BLURBEHIND;

typedef HRESULT(WINAPI * PFN_DwmEnableBlurBehindWindow)(HWND,const DWM_BLURBEHIND*);

void setTransparentBuffer(HWND hwnd)
{
    auto dll = LoadLibraryA("dwmapi.dll");
    auto DwmEnableBlurBehindWindow = (PFN_DwmEnableBlurBehindWindow)GetProcAddress((HMODULE) dll, "DwmEnableBlurBehindWindow");
    HRGN region = CreateRectRgn(0, 0, -1, -1);
    DWM_BLURBEHIND bb = {0};
    bb.dwFlags = DWM_BB_ENABLE | DWM_BB_BLURREGION;
    bb.hRgnBlur = region;
    bb.fEnable = TRUE;

    DwmEnableBlurBehindWindow(hwnd, &bb);
    DeleteObject(region);
}
```

最终使用 QOpenGLWindow，手动设置透明 FrameBuffer，然后设置 `LWA_COLORKEY`，实现了完美的 OpenGL 内容的异形窗口。
