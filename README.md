# 教程：抖音收藏夹 → 结构化知识库

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Public](https://img.shields.io/badge/repo-public-green.svg)](https://github.com/abclq/douyin-kb-tutorial)
[![STT](https://img.shields.io/badge/STT-groq--asr--proxy-blue)](https://github.com/abclq/groq-asr-proxy)

教你如何把抖音收藏的视频，整理成带「文案原文 + 语音逐字稿 + AI 补充洞察」的 Markdown 知识库。
本仓库**只含教程与脚本模板，不含任何真实收藏数据**。

## 🎯 产出效果

每条收藏 → 一个 Markdown 文件：

```markdown
# 视频标题

- 标签：xxx, yyy
- 分类：求职 / 自我提升
- 作者：@xxx  |  点赞：xxx

## 视频文案原文
（完整保留，不摘要）

## 语音逐字稿
（音频转写，完整保留）

## 🤖 AI 补充洞察
- 核心观点：...
- 延伸思考：...
- 可执行建议：...
```

## 🏗 整体架构

```
抖音收藏夹 (Cookie 登录态)
   │  ① 抓取列表 + 文案 + 播放地址
   ▼
本地流水线 (Python + ffmpeg)
   ├─ ② 下载音频 (ffmpeg → 16k wav)
   ├─ ③ 语音转写 ──────────► [groq-asr-proxy 网关]
   │                         Groq → 腾讯云 → 本地硅基流动
   └─ ④ AI 整理 + 补充洞察 ──► 大模型 (如美团/硅基流动对话模型)
   ▼
知识库 (Markdown 文件)
```

## 📋 步骤

### 1. 准备抖音登录态

用浏览器登录抖音，导出 Cookie（或让机器人扫码登录），存为本地文件（**勿入仓**）。
脚本读取 Cookie 调用抖音接口抓取收藏列表。

```python
# cookies.txt 格式: Netscape cookie 格式, 权限 600, 绝不入仓
COOKIES = "/path/to/douyin_cookies.txt"
```

### 2. 抓取收藏列表

```python
import requests
# 通过抖音分享接口拿 aweme_id / 文案 / 播放地址
# 去重后保存为 clean.json: [{aweme_id, author, desc, share_url, play_url, statistics}, ...]
```

### 3. 下载音频 + 转写

```python
import subprocess, requests

def download_audio(play_url, out):
    subprocess.run(["ffmpeg", "-y", "-headers", f"Referer: https://www.douyin.com/\r\n",
                    "-i", play_url, "-vn", "-ac", "1", "-ar", "16000", "-f", "wav", out])

def asr(wav_path):
    # 走 groq-asr-proxy 网关 (见 https://github.com/abclq/groq-asr-proxy)
    with open(wav_path, "rb") as f:
        r = requests.post(STT_URL, headers={"Authorization": "Bearer <AUTH_TOKEN>"},
                          files={"file": f}, data={"language": "zh", "model": "whisper-large-v3-turbo"})
    return r.json().get("text", "")
```

### 4. AI 整理 + 补充洞察

```python
def organize(desc, transcript):
    sys_p = "把抖音收藏整理成知识库笔记。规则：1) 绝不摘要，保留全部原文；2) 文案原文完整保留；3) 逐字稿完整保留；4) 开头给分类标签；5) 用清晰 Markdown 结构。"
    user_p = f"【文案】{desc}\n\n【逐字稿】{transcript}\n\n输出完整 Markdown，末尾追加 ## 🤖 AI 补充洞察 (核心观点/延伸思考/可执行建议)，标注AI生成。"
    # 调大模型 (model=美团 或 硅基流动对话模型)，max_tokens 给足 (32000)
```

### 5. 入库 + 总索引

- 每条存一个 `.md`（文件名用 aweme_id + 标题）
- `add_frontmatter.py`：给所有 md 补 YAML frontmatter（标签/分类/来源），生成总索引 `README` 或 `index.md`

## 🔗 关联项目

- **STT 后端**：[groq-asr-proxy](https://github.com/abclq/groq-asr-proxy) —— 语音转写网关（Groq→腾讯云→本地硅基流动）
- **技术栈总览**：[linqi-ai-stack](https://github.com/abclq/linqi-ai-stack)

## ⚠️ 隐私提醒

- 抖音 Cookie、真实收藏内容 **务必本地保存，绝不提交到任何仓库**
- 本教程仓库只放方法论与脚本模板

## 📄 License

MIT
