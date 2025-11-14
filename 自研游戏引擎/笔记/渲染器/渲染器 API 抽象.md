---
title: 渲染器 API 抽象
date: 2025-11-11
tags:
  - 自研游戏引擎
  - Cpp
category: 游戏引擎
description: 基于 Cherno Hazel 引擎的入门教学，涵盖引擎核心概念、项目搭建、动态链接实践与链接方式对比，适合游戏开发新手
---
---


# 一、抽象目标与核心思路

本次渲染器 API 抽象的核心目标是 **解耦上层逻辑与 OpenGL 依赖**，通过 “抽象基类 + 平台实现 + 工厂函数” 的设计，为后续跨平台扩展（如 Vulkan/DirectX）奠定基础。核心思路遵循 “依赖倒置原则”：上层代码仅依赖统一的抽象接口，底层平台实现（如 OpenGL）适配抽象基类，避免直接耦合。

# 二、已实现的核心抽象组件

## 2.1 渲染 API 标识与全局管理：Renderer 类 + RendererAPI 枚举

## 核心作用

* 标记当前使用的渲染后端（仅支持 OpenGL）；
* 提供全局 API 查询接口，为工厂函数提供分支依据。

## 关键代码总结

```cpp
//Renderer.h
// 渲染API枚举：定义支持的渲染后端（当前仅OpenGL）
enum class RendererAPI
{
    None = 0, OpenGL = 1
};

// 渲染器全局管理类
class Renderer
{
public:
    // 全局获取当前渲染API
    inline static RendererAPI GetAPI() { return s_RendererAPI; }
private:
    // 静态成员：存储当前渲染API
    static RendererAPI s_RendererAPI;
};

// Rederer.cpp 初始化当前渲染API为OpenGL
RendererAPI Renderer::s_RendererAPI = RendererAPI::OpenGL;
```

## 设计亮点

* 枚举扩展灵活：后续添加 Vulkan/DirectX 时，仅需在枚举中新增项（如`Vulkan = 2`）；
* 全局统一管理：上层可通过`Renderer::GetAPI()`感知当前渲染后端，便于条件适配。

## 2.2 缓冲对象抽象：VertexBuffer 与 IndexBuffer 基类

## 核心作用

* 定义缓冲对象的统一接口（绑定 / 解绑），屏蔽 OpenGL 原生调用；
* 通过工厂函数`Create`自动创建对应平台的缓冲实例，上层无需感知具体实现。

## 关键代码总结

### 1. VertexBuffer 基类（顶点缓冲抽象）

```cpp
//Buffer.h
class VertexBuffer
{
public:
    virtual ~VertexBuffer() {};

    // 统一接口：绑定/解绑顶点缓冲
    virtual void Bind() const  = 0;
    virtual void UnBind() const = 0;

    // 工厂函数：根据当前渲染API创建平台对应的VertexBuffer实例
    static VertexBuffer* Create(float* vertices, uint32_t size);
};
```

### 2. IndexBuffer 基类（索引缓冲抽象）

```cpp
//Buffer.h
class IndexBuffer
{
public:
    virtual ~IndexBuffer() {};

    // 统一接口：绑定/解绑索引缓冲
    virtual void Bind() const  = 0;
    virtual void UnBind() const = 0;

    // 统一接口：获取索引数量
    virtual uint32_t GetCount() const = 0;

    // 工厂函数：根据当前渲染API创建平台对应的IndexBuffer实例
    static IndexBuffer* Create(uint32_t* indices, uint32_t size);
};
```

## 设计亮点

* 纯虚接口强制统一：所有平台实现必须遵循`Bind`/`UnBind`等核心接口，保证上层调用一致性；
* 工厂函数隐藏实现：上层通过`VertexBuffer::Create`创建实例，无需手动`new OpenGLVertexBuffer`，降低耦合。

## 2.3 OpenGL 平台实现：OpenGLVertexBuffer 与 OpenGLIndexBuffer

## 核心作用

* 封装 OpenGL 原生缓冲操作（创建 / 绑定 / 数据上传 / 资源释放）；
* 实现`VertexBuffer`/`IndexBuffer`基类接口，适配抽象层。

## 关键代码总结

### 1. OpenGLVertexBuffer（顶点缓冲 OpenGL 实现）

```cpp
//OpenGLBuffer
class OpenGLVertexBuffer : public VertexBuffer
{
public:
    // 构造：创建OpenGL缓冲对象并上传顶点数据
    OpenGLVertexBuffer(float* vertices, uint32_t size);
    virtual ~OpenGLVertexBuffer();

    // 实现基类接口
    virtual void Bind() const override;
    virtual void UnBind() const override;
private:
    // OpenGL缓冲对象ID（平台相关资源）
    uint32_t m_RendererID = 0;
};

// 实现细节：封装glCreateBuffers、glBindBuffer等原生调用
OpenGLVertexBuffer::OpenGLVertexBuffer(float* vertices, uint32_t size)
{
    glCreateBuffers(1, &m_RendererID);
    glBindBuffer(GL_ARRAY_BUFFER, m_RendererID);
    glBufferData(GL_ARRAY_BUFFER, size, vertices, GL_STATIC_DRAW);
}

void OpenGLVertexBuffer::Bind() const { glBindBuffer(GL_ARRAY_BUFFER, m_RendererID); }
void OpenGLVertexBuffer::UnBind() const { glBindBuffer(GL_ARRAY_BUFFER, 0); }
```

### 2. OpenGLIndexBuffer（索引缓冲 OpenGL 实现）

```cpp
//OpenGLBuffer
class OpenGLIndexBuffer : public IndexBuffer
{
public:
    // 构造：创建OpenGL缓冲对象并上传索引数据
    OpenGLIndexBuffer(uint32_t* indices, uint32_t count);
    virtual ~OpenGLIndexBuffer();

    // 实现基类接口
    virtual void Bind() const override;
    virtual void UnBind() const override;
    virtual uint32_t GetCount() const override { return m_Count; }
private:
    uint32_t m_RendererID = 0; // OpenGL缓冲对象ID
    uint32_t m_Count = 0;      // 索引数量（缓存，避免重复计算）
};

// 实现细节：封装OpenGL索引缓冲操作
OpenGLIndexBuffer::OpenGLIndexBuffer(uint32_t* indices, uint32_t count)
    : m_Count(count)
{
    glCreateBuffers(1, &m_RendererID);
    glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, m_RendererID);
    glBufferData(GL_ELEMENT_ARRAY_BUFFER, count * sizeof(uint32_t), indices, GL_STATIC_DRAW);
}

void OpenGLIndexBuffer::Bind() const { glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, m_RendererID); }
void OpenGLIndexBuffer::UnBind() const { glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, 0); }
```

## 设计亮点

* 平台细节完全封装：OpenGL 原生调用仅存在于`Platform/OpenGL`目录下，上层代码无感知；
* 资源自动释放：析构函数中调用`glDeleteBuffers`释放 GPU 资源，避免内存泄漏；
* 索引数量缓存：`m_Count`存储索引总数，避免上层重复计算`sizeof(indices)/sizeof(uint32_t)`。

## 2.4 工厂函数实现：跨平台实例创建入口

## 核心作用

* 根据`Renderer::GetAPI()`自动选择平台实现，创建对应缓冲实例；
* 上层无需关心 “创建哪个平台的缓冲”，只需调用`Create`方法。

## 关键代码总结

```cpp
// VertexBuffer工厂函数
VertexBuffer* VertexBuffer::Create(float* vertices, uint32_t size)
{
    switch (Renderer::GetAPI())
    {
        case RendererAPI::None:    HZ_CORE_ASSERT(false, "不支持None渲染API！");
        case RendererAPI::OpenGL:  return new OpenGLVertexBuffer(vertices, size);
    }

    HZ_CORE_ASSERT(false, "未知的渲染API！")
    return nullptr;
}

// IndexBuffer工厂函数
IndexBuffer* IndexBuffer::Create(uint32_t* indices, uint32_t size)
{
    switch (Renderer::GetAPI())
    {
        case RendererAPI::None:    HZ_CORE_ASSERT(false, "不支持None渲染API！");
        case RendererAPI::OpenGL:  return new OpenGLIndexBuffer(indices, size);
    }

    HZ_CORE_ASSERT(false, "未知的渲染API！")
    return nullptr;
}
```

## 设计亮点

* 扩展成本低：后续添加 Vulkan 实现时，仅需在`switch`中新增分支（`case RendererAPI::Vulkan: return new VulkanVertexBuffer(...)`）；
* 错误防护：通过`HZ_CORE_ASSERT`拦截不支持的 API 或未知 API，便于调试。

## 2.5 Application 上层代码适配：脱离 OpenGL 原生调用

## 核心修改

* 替换原生`glGenBuffers`/`glBindBuffer`等调用，改为使用抽象接口创建缓冲；
* 通过智能指针（`std::unique_ptr`）管理缓冲实例生命周期。

## 关键代码总结

```cpp
// Application构造函数中渲染初始化（仅保留用户提供的代码）

// 顶点数据
float vertices[3 * 3] = {
    -0.5f, -0.5f, 0.0f,
     0.5f, -0.5f, 0.0f,
     0.0f,  0.5f, 0.0f
};

// 索引数据
uint32_t indices[3] = { 0, 1, 2 };

// 创建VAO（暂时保留OpenGL原生调用，后续抽象）
glGenVertexArrays(1, &m_VertexArray);
glBindVertexArray(m_VertexArray);

// 改为通过抽象接口创建VBO（核心适配）
m_VertexBuffer.reset(VertexBuffer::Create(vertices, sizeof(vertices)));

// 暂时保留顶点属性配置（后续抽象）
glEnableVertexAttribArray(0);
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), nullptr);

// 改为通过抽象接口创建IBO（核心适配）
m_IndexBuffer.reset(IndexBuffer::Create(indices, sizeof(indices)/sizeof(uint32_t)));
```

## 适配亮点

* 脱离平台依赖：创建缓冲的逻辑从 “直接调用 OpenGL 函数” 改为 “调用抽象接口”，上层与 OpenGL 解耦；
* 资源管理自动化：使用`std::unique_ptr`管理`VertexBuffer`/`IndexBuffer`实例，自动释放资源，避免内存泄漏。

# 三、当前抽象的核心价值

## 3.1 解耦上层与渲染 API

Application 等上层模块不再直接依赖`glCreateBuffers`、`glBindBuffer`等 OpenGL 原生调用，仅依赖`VertexBuffer`/`IndexBuffer`抽象接口，后续切换渲染后端时无需修改上层代码。

## 3.2 跨平台扩展能力

新增渲染 API（如 Vulkan）时，仅需：

1. 在`RendererAPI`枚举中添加对应项；

2. 实现`VulkanVertexBuffer`/`VulkanIndexBuffer`，适配抽象基类；

3. 在工厂函数中添加新 API 的分支判断；

4. 修改`Renderer::s_RendererAPI`为新 API，上层逻辑完全不变。

## 3.3 代码整洁与可维护性

* 平台相关代码集中在`Platform/OpenGL`目录，便于管理和修改；

* 统一的接口设计（`Bind`/`UnBind`/`Create`）降低上层学习和使用成本；

* 错误防护机制（`HZ_CORE_ASSERT`）减少因 API 使用错误导致的崩溃。

# 四、后续扩展方向（基于当前抽象）

1. 完成`VertexArray`（顶点数组）抽象：替换 Application 中残留的`glGenVertexArrays`等原生调用，实现 VAO 的跨平台支持；

2. 扩展缓冲数据类型：支持动态缓冲（`GL_DYNAMIC_DRAW`）、顶点颜色 / 纹理坐标等复杂数据；

3. 完善错误处理：在平台实现中添加更细致的错误捕获（如`glGetError`），提升调试效率；

4. 支持多渲染 API 动态切换：通过配置文件或编译宏，无需修改代码即可切换渲染后端。

# 总结

当前已完成的渲染器 API 抽象，通过 “标识枚举 + 抽象基类 + 工厂函数 + 平台实现” 的架构，成功实现了上层逻辑与 OpenGL 的解耦，为 Hazel 引擎的跨平台能力打下了核心基础。后续只需基于现有抽象，逐步完善缓冲区布局、顶点数组、着色器、渲染命令等组件的抽象，即可实现完整的跨平台渲染管线。
