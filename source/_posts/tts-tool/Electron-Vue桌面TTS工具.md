---
title: "Electron + Vue 搭建桌面 TTS 工具的技术选型"
date: 2025-06-03 21:30:00
tags:
  - Electron
  - Vue
  - TTS
  - 桌面应用
categories:
  - 开发日志
top_img: false
---

<!-- more -->

`MiMo TTS Studio` 是一个桌面配音工作台，集成小米 MiMo-V2.5-TTS 引擎，支持多角色音色管理、批量并发生成、波形可视化和 WAV/MP3 导出。本文记录技术选型的核心考量。

## 架构概览

{% mermaid %}
graph TD
    subgraph "Electron 主进程"
        M["main.js"] --> C["Controller/tts.js"]
        C --> S["Service/tts.js"]
        S --> API["小米 TTS API"]
        S --> FS["文件系统"]
    end
    subgraph "Vue 渲染进程"
        UI["Vue 3 + Naive UI"] --> IPC["IPC Bridge"]
        IPC --> M
    end
    subgraph "preload"
        BR["bridge.js"] -->|"contextBridge"| UI
    end
{% endmermaid %}

项目基于 `electron-egg` 框架，采用 Controller → Service 的 MVC 分层。前后端通过 IPC 通信，渲染进程通过 `contextBridge` 暴露的 `ipcRenderer` 访问 Node.js API。

## 为什么选 Electron + Vue？

三个核心原因：

**1. 本地文件系统访问**：TTS 工具需要读写音频文件、管理缓存、导出 WAV/MP3。Web 应用无法直接操作文件系统，Electron 通过 Node.js 的 `fs` 模块完全解决这个问题。

**2. 跨平台桌面体验**：一次开发，Windows/macOS/Linux 三端运行。对于工具类应用，桌面体验（窗口管理、系统托盘、本地通知）比 Web 应用更自然。

**3. Vue 3 生态成熟**：Composition API + Pinia 状态管理 + Naive UI 组件库，开发效率高。Vite 做构建工具，热更新速度快。

## IPC 通信：前后端的安全桥接

Electron 的安全模型要求渲染进程不能直接访问 Node.js API。项目通过 `contextBridge` 暴露安全的 IPC 接口：

```javascript
// app/electron/preload/bridge.js — 直接暴露整个 ipcRenderer 对象
const { contextBridge, ipcRenderer } = require('electron')

contextBridge.exposeInMainWorld('electron', {
  ipcRenderer: ipcRenderer,
})
```

渲染进程通过封装的 `invoke` 函数调用主进程，超时机制防止请求挂起：

```javascript
// app/frontend/src/utils/ipcRenderer.js
const Renderer = (window.require && window.require('electron')) || window.electron || {};
const ipc = Renderer.ipcRenderer || undefined;

async function invoke(channel, params, timeout = 60000) {
  if (!ipc?.invoke) {
    throw new Error('IPC not available: not in Electron environment')
  }
  return new Promise((resolve, reject) => {
    const timer = setTimeout(() => {
      reject(new Error(`IPC timeout: ${channel} (${timeout}ms)`))
    }, timeout)
    ipc.invoke(channel, params)
      .then(result => { clearTimeout(timer); resolve(result) })
      .catch(error => { clearTimeout(timer); reject(error) })
  })
}
```

注意：超时默认 60 秒而非 30 秒，且 `bridge.js` 直接暴露整个 `ipcRenderer` 对象，不做方法包装。

## TTS 引擎接入

小米 MiMo-V2.5-TTS 引擎通过 REST API 接入。Service 层封装了 API 调用、重试和缓存（以下为简化示意，实际实现包含完整的错误处理和重试逻辑）：

```javascript
// app/electron/service/tts.js — 简化示意
async _callApi(text, voiceId, speed, styleDescription, exportFormat, apiBase, apiKey, params) {
  // 实际实现包含重试机制、错误处理和音频格式转换
}
```

缓存键用 djb2 哈希算法，由 `_buildCacheKey(text, voiceId, speed, styleDesc, format, useVoiceDesign, useVoiceClone, provider)` 构建。

批量生成时，通过信号量（`_acquireSlot` / `_releaseSlot`）控制并发数，单条失败不影响整体：

```javascript
// frontend/src/views/mimo/store/ttsStore.js — 简化示意
async generateAllLines(charId) {
  // 用信号量控制并发，逐行调用 TTS
  // await _acquireSlot()
  // try { ... } finally { _releaseSlot() }
}
```

## 波形可视化

渲染进程接收音频 buffer 后，用 Web Audio API 解码并绘制波形：

```javascript
const audioContext = new AudioContext()
const audioBuffer = await audioContext.decodeAudioData(buffer)
const channelData = audioBuffer.getChannelData(0)
// 按固定间隔采样，绘制到 Canvas
```

这是纯前端能力，不需要 Node.js 参与，所以直接在渲染进程处理。

## 项目结构

```
electron/
  main.js              # Electron 入口
  preload/
    bridge.js          # contextBridge IPC 桥接
    lifecycle.js       # 窗口生命周期 + IPC handler 注册
  controller/tts.js    # 参数校验 + 错误包装
  service/tts.js       # 业务逻辑（API 调用、缓存、重试）
frontend/
  src/
    api/index.js       # IPC 频道路由表
    utils/ipcRenderer.js  # IPC 封装（超时机制）
    stores/            # Pinia 状态管理
    views/             # Vue 页面组件
```

## 桌面应用的独特挑战

**打包体积**：Electron 应用打包后约 200MB+。通过 `electron-builder` 的 `asar` 压缩和 `files` 白名单排除不必要的文件，控制体积。

**跨平台兼容**：文件路径、字体渲染、音频播放三端表现不同。统一用 `path.join()` 处理路径，音频播放用 Web Audio API 避免平台差异。

**自动更新**：`electron-builder` 内置 `autoUpdater`，配合 GitHub Releases 或自建更新服务器实现静默更新。

> 选型的本质是匹配问题。桌面 + 文件系统 + 跨平台 = Electron 是最直接的解。