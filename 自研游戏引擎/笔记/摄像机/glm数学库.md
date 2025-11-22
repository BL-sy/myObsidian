---
title: 摄像机
date: 2025-11-05
tags:
  - 自研游戏引擎
  - Cpp
  - glm
category: 游戏引擎
description: 集成 GLM 数学库，实现向量、矩阵、四元数等数学工具的封装与应用，为引擎变换、物理、渲染模块提供数学计算支持
---
---


在游戏引擎开发中，渲染模块离不开大量复杂的数学运算，比如向量运算、矩阵变换等。自研数学库不仅需要攻克复杂的算法逻辑，还需优化运算效率以适配实时渲染需求，而 **SIMD**（单指令多数据）技术作为提升计算性能的关键，对开发者的技术储备要求较高。

**GLM**（OpenGL Mathematics）作为一款专为图形开发设计的数学库，不仅封装了完备的数学运算接口，还原生支持 SIMD 加速，能大幅降低开发成本并提升引擎性能。

# 一、GLM 库简介与核心优势

GLM 是一款基于 C++ 的开源数学库，设计理念与 OpenGL 着色器语言（GLSL）高度兼容，可无缝衔接图形渲染流程。其核心优势如下：

1. **功能完备**：涵盖向量（vec2/vec3/vec4）、矩阵（mat2/mat3/mat4）、四元数等图形开发必备的数学结构，同时提供透视投影、平移、旋转、缩放等常用变换接口。
2. **性能优化**：原生支持 SIMD 指令集（如 SSE、AVX），可充分利用 CPU 并行计算能力，相比自研普通数学代码，运算效率提升数倍。
3. **无编译依赖**：采用 “单文件头文件” 设计（所有实现均在.hpp文件中），无需编译生成静态库 / 动态库，仅需通过 “包含目录” 即可引用，简化项目配置。

GLM 官方资源：
- [官方文档](https://glm.g-truc.net/0.9.9/index.html)
- [GitHub 仓库](https://github.com/g-truc/glm)

# 二、在 Hazel 引擎中集成 GLM 的步骤

## 1. 通过 Git 子模块添加 GLM 到项目

为便于后续维护 GLM 版本（如更新、回退），使用 Git 子模块将 GLM 库关联到 Hazel 项目中，具体命令如下：
```bash
# 在项目根目录执行，将GLM克隆到 Hazel/vendor/glm 目录下
git submodule add https://github.com/g-truc/glm Hazel/vendor/glm
```

执行成功后，GLM 的源代码会被存储在Hazel/vendor/glm路径下，且项目的.gitmodules文件会自动记录子模块信息，确保团队协作时依赖一致。

## 2. 修改 Premake 配置文件（premake5.lua）

### （1）定义 GLM 包含目录

首先在配置文件开头定义 GLM 的相对路径，与其他依赖库（如 GLFW、Glad）的目录统一管理：
```lua
-- 定义所有依赖库的包含目录，便于后续复用
IncludeDir = {}
IncludeDir["GLFW"] = "Hazel/vendor/GLFW/include"
IncludeDir["Glad"] = "Hazel/vendor/Glad/include"
IncludeDir["ImGui"] = "Hazel/vendor/imgui"
IncludeDir["glm"] = "Hazel/vendor/glm"  -- GLM库的根目录（包含glm文件夹）
```

### （2）关联 Hazel 项目与 GLM

在project "Hazel"（引擎核心项目）中，需添加 GLM 的头文件到 “文件列表”，并将 GLM 目录加入 “编译器包含目录”，确保编译器能找到 GLM 的.hpp文件：
```lua
-- 包含相对解决方案的目录
IncludeDir = {}
IncludeDir["glm"] = "Hazel/vendor/glm"
    
project "Hazel"  --Hazel项目
    ......
    -- 包含的所有h和cpp文件
    files{
        "%{prj.name}/src/**.h",
        "%{prj.name}/src/**.cpp",
        -- 包含hpp文件
        "%{prj.name}/vendor/glm/glm/**.hpp",
        "%{prj.name}/vendor/glm/glm/**.inl"
    }
    -- 包含目录
    includedirs{
        "%{IncludeDir.glm}"-- 包含目录
    }
```

### （3）配置 Sandbox 项目（测试工程）

Sandbox 作为 Hazel 的测试项目，同样需要引用 GLM，需在其includedirs中添加 GLM 目录：
```lua
-- 配置Sandbox测试项目
project "Sandbox"

    -- 配置包含目录：添加GLM目录
    includedirs{
        "%{IncludeDir.glm}"             -- GLM目录（关键配置）
    }
```

# 三、GLM 功能测试：实现相机矩阵变换

为验证 GLM 的集成效果，我们在 Sandbox 项目的SandboxApp.cpp中编写相机矩阵变换逻辑 —— 通过 GLM 实现透视投影、视图变换（平移 + 旋转）和模型缩放，最终生成完整的 MVP（Model-View-Projection）矩阵，这是 3D 渲染中确定物体最终屏幕位置的核心计算流程。
## 1. 测试代码实现

```cpp
#include <Hazel.h>  // 引入Hazel引擎核心头文件

// 引入GLM的核心头文件
#include <glm/vec3.hpp>               // 3D向量（vec3）
#include <glm/vec4.hpp>               // 4D向量（vec4，用于齐次坐标）
#include <glm/mat4x4.hpp>             // 4x4矩阵（mat4，图形变换核心）
#include <glm/gtc/matrix_transform.hpp>  // 矩阵变换工具（平移、旋转、缩放、透视）

/**
 * 生成相机的MVP矩阵
 * @param Translate：相机沿Z轴的平移距离（控制远近）
 * @param Rotate：相机旋转角度（x轴：左右旋转，y轴：上下旋转）
 * @return 完整的MVP矩阵（Projection * View * Model）
 */
glm::mat4 camera(float Translate, glm::vec2 const& Rotate)
{
    // 1. 透视投影矩阵：模拟人眼视角，近裁剪面0.1f，远裁剪面100.f
    glm::mat4 Projection = glm::perspective(
        glm::radians(45.0f),  // 垂直视场角（45度，需转换为弧度）
        4.0f / 3.0f,          // 宽高比（4:3，可根据窗口尺寸动态调整）
        0.1f,                 // 近裁剪面（距离相机过近的物体不渲染）
        100.0f                // 远裁剪面（距离相机过远的物体不渲染）
    );

    // 2. 视图矩阵：模拟相机的位置和朝向（平移+旋转）
    glm::mat4 View = glm::mat4(1.0f);  // 初始化单位矩阵
    // 步骤1：沿Z轴负方向平移（Translate越大，相机离物体越远）
    View = glm::translate(View, glm::vec3(0.0f, 0.0f, -Translate));
    // 步骤2：绕X轴旋转（Rotate.y控制上下仰俯，负号调整旋转方向）
    View = glm::rotate(View, Rotate.y, glm::vec3(-1.0f, 0.0f, 0.0f));
    // 步骤3：绕Y轴旋转（Rotate.x控制左右旋转）
    View = glm::rotate(View, Rotate.x, glm::vec3(0.0f, 1.0f, 0.0f));

    // 3. 模型矩阵：控制物体自身的缩放（此处将物体缩小为原来的50%）
    glm::mat4 Model = glm::scale(
        glm::mat4(1.0f),  // 初始化单位矩阵
        glm::vec3(0.5f)   // x/y/z轴均缩放0.5倍
    );

    // 4. 返回MVP矩阵：投影矩阵 * 视图矩阵 * 模型矩阵（顺序不可颠倒）
    return Projection * View * Model;
}

// 自定义Layer，用于测试GLM功能
class ExampleLayer : public Hazel::Layer
{
public:
    ExampleLayer()
        : Layer("Example")  // 初始化Layer名称
    {
        // 在Layer构造时调用camera函数，测试矩阵生成
        glm::mat4 mvp = camera(3.0f, { 2.0f, 1.0f });
    }
};
```

## 2. 代码说明与验证

- **透视投影（Projection）**：通过glm::perspective生成，参数中的 “视场角” 和 “宽高比” 决定了相机的视野范围，“近 / 远裁剪面” 则控制渲染的深度范围，避免无效计算。
- **视图变换（View）**：通过glm::translate和glm::rotate组合，模拟相机的移动和旋转 —— 例如Translate=3.0f表示相机沿 Z 轴负方向移动 3 个单位，与物体保持 3 个单位的距离。
- **模型变换（Model）**：通过glm::scale缩小物体，确保物体在屏幕内正常显示。
- **验证方式**：编译并运行 Sandbox 项目，调试时观察mvp矩阵的值不为空，则说明 GLM 集成成功且功能正常。


通过以上步骤，我们已成功将 GLM 数学库集成到 Hazel 引擎中，并实现了核心的矩阵变换功能。GLM 的引入不仅规避了自研数学库的复杂度，还通过 SIMD 加速提升了渲染计算效率，为后续 Hazel 引擎的 3D 渲染模块（如顶点变换、光照计算）奠定了基础。