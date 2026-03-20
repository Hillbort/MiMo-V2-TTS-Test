# MiMo-V2-TTS-Test
# MiMo-V2-TTS API 接入测试全记录

> **测试时间**：2026年3月19日 17:30 - 23:40
> **测试目标**：基于小米 MiMo-V2-TTS 开放 API，开发一个完整的语音合成 Web 前端页面
> **测试者**：普通开发者
> **报告撰写**：MiMo-V2-Pro
> **API 地址**：`https://api.xiaomimimo.com/v1/chat/completions`
> **模型**：`mimo-v2-tts`

---

## 一、背景与目标

小米开放了 MiMo-V2-TTS 语音合成大模型的 API，官网提供了 OpenAI 兼容接口文档。本次测试的目标是：

1. 调通 API，实现基本的文本转语音功能
2. 验证风格控制、方言、角色扮演、声音事件等高级功能
3. 开发一个开箱即用的单页 Web 应用
4. 记录完整的踩坑过程，为其他开发者提供参考

---

## 二、测试过程总览

| 轮次 | 时间 | 操作 | 结果 | 问题 |
|------|------|------|------|------|
| 第1轮 | 17:35 | 构建完整前端 + Node.js 代理服务器 | ❌ 报错 | 浏览器 CORS 跨域限制 |
| 第2轮 | 17:39 | 改为纯前端直连 API，messages 用 `user` + `assistant` | ❌ 合成内容错误 | assistant 消息放在 user 后面，模型合成了 assistant 的内容而非 user 的文本 |
| 第3轮 | 17:43 | 去掉 assistant，只保留 user 消息 | ❌ 400 报错 | API 返回 `"system role is not allowed for TTS model"`（实际是用了 system 角色） |
| 第4轮 | 23:34 | 只保留 user 消息，不加 system/assistant | ❌ 400 报错 | API 返回 `"messages must contain an assistant role for TTS model"` |
| 第5轮 | 23:37 | assistant 放前面 + `[default style]`，user 放后面 | ⚠️ 部分成功 | 200 返回了音频，但风格标签格式不对 |
| 第6轮 | 23:40 | 查阅官方文档，assistant 用 `<style>` 标签包裹文本 | ✅ 成功 | 合成正确，音频完整 |

---

## 三、逐轮详细记录

### 第1轮：CORS 跨域问题

**操作**：构建了 Node.js Express 代理服务器 + 前端页面，通过代理中转请求。

**日志**：
```
[17:35:01] → POST /v1/audio/speech    ← 端点写错了
[17:35:02] ← Failed to fetch          ← 浏览器直接拦截
```

**问题分析**：
- 浏览器安全策略禁止前端页面直接请求不同域名的 API（CORS 跨域限制）
- 同时 API 端点写成了 `/v1/audio/speech`，不是官方的 `/v1/chat/completions`

**解决方案**：
- 方案A：搭建代理服务器（复杂，不适合小白）
- 方案B：改为纯前端，依赖用户自行解决 CORS（推荐，最终采用）

---

### 第2轮：messages 顺序搞反

**操作**：改为纯前端，使用 `/v1/chat/completions` 端点，messages 结构如下：

```json
{
  "messages": [
    {"role": "user",      "content": "你好，我是小米 MiMo..."},
    {"role": "assistant",  "content": "[用自然平和的风格说]"}
  ]
}
```

**日志**：
```
[17:39:07] → POST /v1/chat/completions
[17:39:09] ← 200
[17:39:10] Base64 音频: 133180 chars
[17:39:10] Tokens: p=154 c=15 t=169
```

**问题**：200 成功返回了音频，但播放后发现**只有短短一个词**，不是输入的完整文本。

**原因分析**：API 返回了 200，说明请求格式基本正确。但模型**合成了 assistant 消息的内容**（`[用自然平和的风格说]`），而不是 user 消息的文本。assistant 消息放在 user 后面，模型认为 assistant 就是要合成的内容。

---

### 第3轮：去掉 assistant，用 system 角色

**操作**：把风格指令放到 system 消息里：

```json
{
  "messages": [
    {"role": "system",  "content": "请用以下风格朗读：自然平和"},
    {"role": "user",    "content": "你好，我是小米 MiMo..."}
  ]
}
```

**日志**：
```
[17:43:31] ← 400
[17:43:43] Error: "messages[0] system role is not allowed for TTS model"
```

**问题**：TTS 模型**明确不支持 system 角色**。错误信息非常清晰。

---

### 第4轮：只保留 user 消息

**操作**：去掉 system 和 assistant，只留 user：

```json
{
  "messages": [
    {"role": "user", "content": "你好，我是小米 MiMo..."}
  ]
}
```

**日志**：
```
[23:34:43] ← 400
[23:34:43] Error: "messages must contain an assistant role for TTS model"
```

**问题**：API 又报错了——**必须包含 assistant 角色**。

**关键发现**：
- ❌ 不支持 system 角色
- ❌ 不能没有 assistant 角色
- ❌ assistant 不能放在 user 后面（会被当成合成内容）
- → 只剩一种可能：assistant 放前面，user 放后面

---

### 第5轮：assistant 放前面

**操作**：

```json
{
  "messages": [
    {"role": "assistant", "content": "[default style]"},
    {"role": "user",      "content": "你好，我是小米 MiMo..."}
  ]
}
```

**日志**：
```
[23:37:25] ← HTTP 200
[23:37:28] 音频来源: message.audio.data (225340 chars)
```

**结果**：200 成功，拿到了音频数据。但风格标签格式 `[default style]` 不是官方格式。

---

### 第6轮：查阅官方文档，修正格式 ✅

**操作**：参考官方文档 `https://platform.xiaomimimo.com/#/docs/usage-guide/speech-synthesis`，使用正确的 `<style>` 标签格式：

```json
{
  "model": "mimo-v2-tts",
  "messages": [
    {"role": "user",      "content": "请帮我合成以下内容"},
    {"role": "assistant",  "content": "<style>开心 东北话</style>哎呀妈呀，这天儿也忒冷了吧！"}
  ],
  "audio": {"format": "wav", "voice": "mimo_default"}
}
```

**结果**：✅ 合成成功，音频完整、风格正确。

---

## 四、API 调用规范总结

### 4.1 请求格式

```
POST https://api.xiaomimimo.com/v1/chat/completions
Authorization: Bearer sk-xxxxxxxxxx
Content-Type: application/json
```

### 4.2 请求体结构

```json
{
  "model": "mimo-v2-tts",
  "messages": [
    {"role": "user",      "content": "请帮我合成以下内容"},
    {"role": "assistant",  "content": "<style>风格1 风格2</style>要合成的文本内容"}
  ],
  "audio": {
    "format": "wav",
    "voice": "mimo_default"
  }
}
```

### 4.3 关键规则

| 规则 | 说明 |
|------|------|
| **必须有 assistant 角色** | 没有会返回 400 |
| **不能有 system 角色** | 有会返回 400 |
| **assistant 放 user 前面** | 放后面会合成 assistant 内容而非 user 文本 |
| **风格用 `<style>` 标签** | 写在 assistant 消息开头，如 `<style>开心 东北话</style>` |
| **多个风格用空格分隔** | `<style>开心 男声 中年 粗犷嗓音 东北话</style>` |
| **风格支持任意自然语言** | `<style>中年男人 粗犷嗓音 刚睡醒</style>` |
| **user 消息可选** | 建议保留，用于调整语气 |

### 4.4 音频标签

直接写在文本中，用括号包裹：

| 标签 | 作用 |
|------|------|
| `(笑声)` | 插入笑声 |
| `(咳嗽)` | 插入咳嗽声 |
| `(停顿2秒)` | 停顿 2 秒 |
| `(叹气)` | 插入叹气声 |
| `(提高音量)` | 后续文本音量增大 |
| `(小声)` | 后续文本音量减小 |
| `(语速加快)` | 后续文本加速 |
| `(语速变慢)` | 后续文本减速 |

### 4.5 音色选项

| voice 值 | 说明 |
|-----------|------|
| `mimo_default` | 默认音色 |
| `default_zh` | 中文女声 |
| `default_en` | 英文女声 |

### 4.6 返回格式

```json
{
  "id": "************************************",
  "choices": [{
    "message": {
      "audio": {
        "data": "UklGRiQAAABXQVZFZm10IBAAAAABAAEA..."
      }
    },
    "finish_reason": "stop"
  }],
  "usage": {
    "prompt_tokens": 154,
    "completion_tokens": 15,
    "total_tokens": 169
  }
}
```

音频在 `choices[0].message.audio.data` 中，是 Base64 编码的 WAV 数据。

---

## 五、踩坑总结

| 序号 | 坑 | 错误信息 | 解决方式 |
|------|------|----------|----------|
| 1 | CORS 跨域 | `Failed to fetch` | 浏览器直连（需用户自行处理 CORS） |
| 2 | API 端点写错 | `Failed to fetch` | 改为 `/v1/chat/completions` |
| 3 | assistant 放 user 后面 | 200 但音频只有几个字 | assistant 放 user 前面 |
| 4 | 用了 system 角色 | `"system role is not allowed for TTS model"` | 去掉 system |
| 5 | 没有 assistant 角色 | `"must contain an assistant role for TTS model"` | 必须保留 assistant |
| 6 | 风格标签格式错误 | 200 但风格不生效 | 用 `<style>xxx</style>` 官方格式 |

---

## 六、最终效果

### 6.1 功能覆盖

-  基本文本转语音合成
-  自然语言风格控制（任意描述）
-  方言支持（东北话、四川话、粤语等）
-  角色扮演（孙悟空、林黛玉等）
-  发音风格（开心、悲伤、悄悄话等）
-  音色特征（男声/女声/中年/老年/粗犷/温柔等）
-  声音事件标签（笑声、咳嗽、停顿等）
-  唱歌模式
-  音频播放、下载、进度条
-  历史记录回放
-  请求日志调试
-  请求预览（实时查看将要发送的内容）

### 6.2 页面功能截图说明

**左侧**：文本输入区 + 风格多选组合（发音风格 / 音色特征 / 方言 / 角色扮演 / 自定义描述 / 音频标签 / 合成模式）

**右侧**：API 设置（Key / 地址 / 模型）+ 参数（音色 / 格式）+ 请求预览 + 合成按钮

**底部**：音频播放器 + 下载按钮 + Token 用量 + 请求日志 + 历史记录

### 6.3 合成效果

最终测试：输入"你好，我是小米 MiMo 语音合成大模型"，选择"开心 + 东北话"风格，合成成功，音频大小约 200KB，播放完整流畅。

---

## 七、给其他开发者的建议

1. **messages 格式是最大的坑**——assistant 必须在 user 前面，风格用 `<style>` 标签
2. **CORS 是浏览器特有的问题**——如果用 Python / curl / Node.js 后端直接调用，不存在这个问题
3. **风格标签支持任意自然语言**——不限于预设列表，你可以写"刚睡醒 迷迷糊糊"、"播音腔 字正腔圆"
4. **打开请求日志**——遇到问题时，日志是最有用的调试工具
5. **Token 用量很少**——一次合成大约消耗 150-200 tokens，非常经济

---

## 八、完整代码

最终版为**单个 HTML 文件**，约 500 行代码，双击即可打开使用，无需安装任何依赖。代码包含完整的 UI 界面、API 调用逻辑、音频播放器、历史记录和调试日志。
