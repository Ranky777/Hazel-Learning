# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build & Run

```bash
# Regenerate Visual Studio project files
./GenerateProjects.bat
# Or directly: vendor/bin/premake/premake5.exe vs2022

# Build via MSBuild (example)
msbuild Hazel.sln /p:Configuration=Debug

# Build via Visual Studio
# Open Hazel.sln and build. Start project is Sandbox.
```

Configurations: Debug, Release, Dist.

## Project Structure

Two projects managed by Premake5 (`premake5.lua`):

- **Hazel** (`Hazel/`) — Static library (C++17). The engine itself. Built with `/EHsc`, precompiled header `hzpch.h`.
- **Sandbox** (`Sandbox/`) — Console application consuming Hazel as a static lib. The "game" / test harness.

Vendors (git submodules): spdlog (logging), GLFW (windowing), Glad (OpenGL loader), ImGui (UI), glm (math), stb_image (texture loading).

Precompiled header: `Hazel/src/hzpch.h` → `hzpch.cpp`.

## High-Level Architecture

### Engine → Client Pattern
- `Hazel::Application` is the engine's main loop. Client defines a `CreateApplication()` function (declared in `EntryPoint.h`) that returns a new Application subclass.
- Entry point is `main()` in `EntryPoint.h` — clients only need `#include <Hazel.h>` and define `CreateApplication()`.

### Layer System
- `Layer` base class: `OnAttach()`, `OnDetach()`, `OnUpdate(Timestep)`, `OnImGuiRender()`, `OnEvent(Event&)`.
- `LayerStack` manages layers as an ordered vector with a split insert index (regular layers at `m_LayerInsertIndex`, overlays pushed after).
- Layers are pushed via `Application::PushLayer()` / `PushOverlay()`.

### Event System
- Blocking event dispatch. Base `Event` class with `Handled` flag.
- `EventDispatcher` dispatches by type: `dispatcher.Dispatch<WindowCloseEvent>(fn)`.
- Event categories: Application, Input, Keyboard, Mouse, MouseButton.
- Concrete events: `WindowResizeEvent`, `WindowCloseEvent`, `AppTickEvent`, `AppUpdateEvent`, `AppRenderEvent`, `KeyPressedEvent`, `KeyReleasedEvent`, `KeyTypedEvent`, `MouseMovedEvent`, `MouseScrolledEvent`, `MouseButtonPressedEvent`, `MouseButtonReleasedEvent`.

### Input
- `Hazel::Input` — static frontend, singleton pattern (`s_Instance`). Platform implementation via `WindowsInput`.
- Key codes from GLFW, defined in `KeyCodes.h` as `HZ_KEY_*`.

### Renderer Abstractions
Abstract interface / platform-specific implementation pattern:

| Abstract (Hazel/Renderer/) | Platform (Platform/OpenGL/) |
|---|---|
| `RendererAPI` — `SetClearColor`, `Clear`, `DrawIndexed` | `OpenGLRendererAPI` |
| `Shader` — `Bind`, `Unbind`, `Create(src1, src2)` | `OpenGLShader` (+ uniform uploaders) |
| `VertexBuffer` / `IndexBuffer` — `Create(verts, size)` | `OpenGLVertexBuffer` / `OpenGLIndexBuffer` |
| `VertexArray` — `AddVertexBuffer`, `SetIndexBuffer` | `OpenGLVertexArray` |
| `Texture` / `Texture2D` | `OpenGLTexture2D` |
| `GraphicsContext` — `Init`, `SwapBuffers` | `OpenGLContext` |

- `RenderCommand` — thin static wrapper around `RendererAPI` singleton.
- `Renderer` — manages scene (BeginScene/EndScene with orthographic camera), queues `Submit` calls.
- `BufferLayout` / `BufferElement` — describes vertex attribute layout with `ShaderDataType`.
- Factory pattern: static `Create()` on abstract classes returns platform-specific instances.

### Window
- Abstract `Window` interface → `WindowsWindow` (GLFW).
- `WindowData` struct holds title, size, VSync flag, and `EventCallbackFn`.
- `WindowsWindow` owns the GLFWwindow and `GraphicsContext`.

### Camera
- `OrthographicCamera` — projection + view matrix, position + rotation.

### Core Utilities
- `Core.h`: `HAZEL_API` dllimport/dllexport macro, `BIT(x)`, `HZ_BIND_EVENT_FN(fn)`, `HZ_(CORE_)ASSERT`, `Scope<T>` = `unique_ptr`, `Ref<T>` = `shared_ptr`.
- `Timestep`: wraps delta time in seconds (implicitly convertible to float).
- Logging via `spdlog`: `HZ_CORE_*` and `HZ_*` macros for trace/info/warn/error/fatal.

### Platform Abstractions
Currently Windows-only. Platform-specific code in `Platform/Windows/` and `Platform/OpenGL/`. Guarded by `#ifdef HZ_PLATFORM_WINDOWS`.

### Key Defines
- `HZ_PLATFORM_WINDOWS` — platform target
- `HZ_BUILD_DLL` — set when building Hazel as DLL (currently static lib, but macro infrastructure exists)
- `HZ_DEBUG` / `HZ_RELEASE` / `HZ_DIST` — config selection
- `HAZEL_API` — dllexport/dllimport
