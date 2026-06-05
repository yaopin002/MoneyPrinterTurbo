# MoneyPrinterTurbo 工程架构与开发指南

> 适用对象：初学者 | 最后更新：2026-06-03

---

## 一、这是什么项目？

**MoneyPrinterTurbo** 是一个"AI 短视频自动生成器"。你只需要给一个**主题**（比如"春天的花海"），它就会自动帮你完成以下所有事情：

1. 写视频文案（调用 AI 大模型）
2. 找视频素材（从 Pexels/Pixabay 下载高清无版权视频）
3. 生成配音（TTS 语音合成）
4. 生成字幕
5. 配背景音乐
6. 把所有素材合成一个完整的短视频

---

## 二、项目结构总览

```
MoneyPrinterTurbo/
├── main.py                 # 【入口1】API 服务启动入口
├── pyproject.toml           # Python 项目依赖配置
├── requirements.txt         # pip 依赖（兼容旧方式）
├── config.example.toml      # 配置文件模板
├── config.toml              # 实际配置文件（从模板复制）
│
├── app/                     # 【核心代码】MVC 架构
│   ├── asgi.py              # FastAPI 应用初始化
│   ├── router.py            # API 路由注册
│   │
│   ├── config/              # 配置模块
│   │   ├── __init__.py
│   │   └── config.py        # 读取 config.toml，环境检测
│   │
│   ├── controllers/         # 控制器层（路由处理）
│   │   ├── v1/
│   │   │   ├── video.py     # 视频任务 CRUD API
│   │   │   └── llm.py       # 文案/术语/社交媒体元数据 API
│   │   └── manager/         # 任务队列管理
│   │       ├── base_manager.py   # 任务管理器基类
│   │       ├── memory_manager.py # 内存版任务管理器
│   │       └── redis_manager.py  # Redis 版任务管理器
│   │
│   ├── models/              # 模型层（数据定义）
│   │   ├── schema.py        # Pydantic 请求/响应模型
│   │   ├── const.py         # 常量定义（任务状态等）
│   │   └── exception.py     # 自定义异常
│   │
│   ├── services/            # 服务层（业务逻辑）
│   │   ├── task.py          # 【核心编排】任务执行流水线
│   │   ├── llm.py           # AI 大模型调用（文案生成）
│   │   ├── video.py         # 视频合成（MoviePy）
│   │   ├── voice.py         # 语音合成（TTS）
│   │   ├── material.py      # 视频素材下载（Pexels/Pixabay）
│   │   ├── subtitle.py      # 字幕生成
│   │   ├── state.py         # 任务状态管理
│   │   └── upload_post.py   # 自动发布到 TikTok/Instagram
│   │
│   └── utils/               # 工具函数
│       ├── utils.py
│       └── file_security.py # 文件路径安全检查
│
├── webui/                   # 【入口2】Web 界面（Streamlit）
│   ├── Main.py              # Streamlit 主页面
│   ├── i18n/                # 国际化翻译文件
│   └── .streamlit/
│       └── webui.toml
│
├── resource/                # 资源文件
│   ├── songs/               # 背景音乐
│   └── fonts/               # 字幕字体
│
├── storage/                 # 运行时生成（自动创建）
│   └── tasks/               # 每个任务的输出文件
│
├── docker-compose.yml       # Docker 部署配置
├── Dockerfile
└── test/                    # 测试文件
```

---

## 三、架构模式：MVC

项目采用 **MVC（Model-View-Controller）** 架构：

| 层级 | 作用 | 对应目录 |
|------|------|----------|
| **Model（模型）** | 定义数据结构，如请求参数、响应格式 | `app/models/` |
| **View（视图）** | 用户界面 | `webui/`（Streamlit）或 API 返回的 JSON |
| **Controller（控制器）** | 接收请求，调用 Service 处理，返回响应 | `app/controllers/` |
| **Service（服务）** | 核心业务逻辑，MVC 中没有严格对应的层，但这里用 Service 层封装 | `app/services/` |

### 请求处理流程

```
用户请求
    │
    ▼
Controller（控制器）
    │ 接收参数、校验、调用 Service
    ▼
Service（服务层）
    │ 执行核心业务逻辑
    │
    ├── llm.py       → 调用 AI 生成文案
    ├── voice.py     → TTS 生成配音
    ├── material.py  → 下载视频素材
    ├── subtitle.py  → 生成字幕
    └── video.py     → 合成最终视频
    │
    ▼
Model（模型）
    │ 封装响应数据格式
    ▼
返回给用户
```

---

## 四、两大入口

### 入口 1：API 服务（FastAPI）

**文件：** `main.py`

```python
import uvicorn
from app.config import config

if __name__ == "__main__":
    uvicorn.run(
        app="app.asgi:app",        # 启动 FastAPI 应用
        host=config.listen_host,   # 默认 0.0.0.0
        port=config.listen_port,   # 默认 8080
    )
```

启动后提供 RESTful API：
- `POST /videos` — 创建视频生成任务
- `GET /tasks/{task_id}` — 查询任务状态
- `DELETE /tasks/{task_id}` — 删除任务
- `POST /scripts` — 生成视频文案
- `POST /terms` — 生成视频关键词
- `POST /social-metadata` — 生成社交媒体发布文案
- `GET /musics` — 获取背景音乐列表
- `POST /musics` — 上传背景音乐
- `GET /video_materials` — 获取本地视频素材列表
- `POST /video_materials` — 上传视频素材
- 访问 `http://127.0.0.1:8080/docs` 可查看 Swagger API 文档

### 入口 2：Web 界面（Streamlit）

**文件：** `webui/Main.py`

这是一个图形化界面，通过 Streamlit 框架构建。它直接调用 `app/services/` 中的业务函数，实现可视化操作。

启动后访问 `http://127.0.0.1:8501`

### 关系图

```
┌─────────────────────────────────────────────────────┐
│                   用户                               │
└──────────┬──────────────────────────┬────────────────┘
           │                          │
           ▼                          ▼
┌──────────────────────┐  ┌──────────────────────────┐
│   API 服务 (FastAPI) │  │   Web 界面 (Streamlit)   │
│   localhost:8080     │  │   localhost:8501          │
│   RESTful API        │  │   可视化表单              │
└──────────┬───────────┘  └──────────┬────────────────┘
           │                          │
           └──────────┬───────────────┘
                      ▼
           ┌──────────────────────┐
           │    Service 服务层     │
           │  (app/services/)     │
           │  业务逻辑核心         │
           └──────────────────────┘
```

---

## 五、核心技术栈

| 技术 | 用途 |
|------|------|
| Python 3.11 | 编程语言 |
| FastAPI | API 框架 |
| Streamlit | Web UI 框架 |
| MoviePy | 视频编辑合成 |
| edge-tts | 免费语音合成（Azure TTS V1） |
| faster-whisper | 本地的语音转文字（用于字幕时间戳） |
| OpenAI SDK | 调用大模型生成文案 |
| Pexels/Pixabay API | 获取免费视频素材 |
| Uvicorn | Python ASGI 服务器 |
| Docker | 容器化部署 |

---

## 六、核心业务流程详解

当用户提交一个视频生成请求时，后台的任务流水线（`app/services/task.py`）依次执行以下步骤：

```
步骤 1: 生成文案 (generate_script)
    │ 调用 LLM（如 GPT-4o）根据主题写一段短视频脚本
    │ 如果用户提供了自定义文案，则跳过此步
    ▼
步骤 2: 生成搜索关键词 (generate_terms)
    │ 根据文案，让 LLM 提取 5 个搜索关键词
    │ 用于后续从 Pexels/Pixabay 搜索视频素材
    ▼
步骤 3: 生成配音音频 (generate_audio)
    │ 使用 TTS（Edge TTS / Azure TTS）将文案转为语音
    │ 同时记录每句话的时间戳（用于字幕对齐）
    ▼
步骤 4: 下载视频素材 (material.py)
    │ 用关键词从 Pexels/Pixabay 搜索并下载视频片段
    │ 也可以使用用户上传的本地素材
    ▼
步骤 5: 生成字幕 (generate_subtitle)
    │ 根据 TTS 返回的时间戳生成 .srt 字幕文件
    │ 或者用 faster-whisper 重新识别生成更精确的字幕
    ▼
步骤 6: 合成最终视频 (video.py → combine_final_video)
    │ 使用 MoviePy 将以下元素合成：
    │   - 视频素材（多个片段拼接）
    │   - 配音音频
    │   - 字幕（叠加在画面上）
    │   - 背景音乐
    │   - 转场效果
    ▼
步骤 7: 完成
    │ 视频文件保存在 storage/tasks/{task_id}/ 目录
    │ 用户可通过 API 或 WebUI 下载
```

---

## 七、配置文件说明

配置文件 `config.toml`（从 `config.example.toml` 复制）主要包含：

```toml
[app]
# LLM 提供商: openai, aihubmix, deepseek, gemini, ollama...
llm_provider = "openai"
openai_api_key = "sk-..."       # 你的 API Key
openai_model_name = "gpt-4o-mini"

# 视频素材来源
pexels_api_keys = []            # 从 pexels.com 申请
pixabay_api_keys = []           # 从 pixabay.com 申请

# 字幕生成方式: edge 或 whisper
subtitle_provider = "edge"

[whisper]                       # whisper 配置（GPU/CPU）
model_size = "large-v3"
device = "CPU"

[azure]                         # Azure TTS V2（付费版）
speech_key = ""
speech_region = ""

[proxy]                         # 代理设置（可选）
# http = "http://proxy:1234"
```

---

## 八、如何从零搭建类似项目？

如果你想**搭建一个类似的项目**（AI 视频生成），以下是推荐的学习路线和搭建步骤：

### 阶段 1：基础知识准备（2-4 周）

```
□ Python 基础: 变量、函数、类、模块
  → 推荐: Python 官方教程 / 廖雪峰 Python 教程

□ FastAPI 基础:
  → 官方文档: https://fastapi.tiangolo.com/zh/tutorial/
  → 重点: 路径操作、依赖注入、Pydantic 模型

□ Streamlit 基础（如果需要 WebUI）:
  → 官方文档: https://docs.streamlit.io/
  → 重点: st.write, st.button, st.selectbox, st.file_uploader

□ MoviePy 基础（视频编辑）:
  → 官方文档: https://zulko.github.io/moviepy/
  → 重点: VideoFileClip, AudioFileClip, CompositeVideoClip, TextClip
```

### 阶段 2：环境搭建

```bash
# 1. 安装 Python 3.11
#    下载地址：https://www.python.org/downloads/

# 2. 安装 uv（推荐，比 pip 快很多）
#    Windows (PowerShell):
#    powershell -c "irm https://astral.sh/uv/install.ps1 | iex"
#    
#    Mac / Linux:
#    curl -LsSf https://astral.sh/uv/install.sh | sh

# 3. 克隆项目并安装依赖
git clone https://github.com/your-username/your-project.git
cd your-project
uv python install 3.11
uv sync --frozen

# 4. 确认 ffmpeg 已安装
ffmpeg -version
# 如果没有，下载地址：https://ffmpeg.org/download.html
```

### 阶段 3：项目脚手架搭建

按照 MVC 模式，创建项目骨架：

```
your-project/
├── main.py                    # FastAPI 启动入口
├── pyproject.toml             # 项目依赖
├── config.toml                # 配置文件
│
├── app/
│   ├── __init__.py
│   ├── asgi.py                # FastAPI 应用工厂
│   ├── router.py              # 路由汇总
│   │
│   ├── config/
│   │   ├── __init__.py
│   │   └── config.py          # 读取配置
│   │
│   ├── controllers/
│   │   ├── __init__.py
│   │   └── v1/                # API v1 版本
│   │       ├── __init__.py
│   │       ├── base.py        # 路由基类（认证等）
│   │       └── video.py       # 视频相关 API
│   │
│   ├── models/
│   │   ├── __init__.py
│   │   └── schema.py          # Pydantic 数据模型
│   │
│   ├── services/
│   │   ├── __init__.py
│   │   ├── task.py            # 任务编排
│   │   ├── llm.py             # AI 大模型调用
│   │   ├── video.py           # 视频合成
│   │   ├── voice.py           # 语音合成
│   │   └── material.py        # 素材下载
│   │
│   └── utils/
│       ├── __init__.py
│       └── utils.py           # 工具函数
│
├── webui/
│   └── Main.py                # Streamlit 界面
│
└── resource/
    ├── songs/                 # 背景音乐
    └── fonts/                 # 字体文件
```

### 阶段 4：核心功能实现顺序

```
第 1 步: 配置系统
  → 实现 config.py，读取 config.toml
  → 接入各种 API Key

第 2 步: LLM 服务
  → 实现 llm.py
  → 调用 OpenAI API 生成文案
  → 功能: generate_script()、generate_terms()

第 3 步: TTS 语音服务
  → 实现 voice.py
  → 集成 edge-tts（免费，无需 API Key）
  → 生成配音音频文件

第 4 步: 视频素材服务
  → 实现 material.py
  → 接入 Pexels/Pixabay API
  → 根据关键词搜索并下载视频片段

第 5 步: 视频合成
  → 实现 video.py
  → 使用 MoviePy 拼接视频 + 配音 + 字幕 + BGM

第 6 步: API 接口
  → 实现 controllers/v1/video.py
  → 创建任务、查询状态、下载结果

第 7 步: WebUI
  → 实现 webui/Main.py
  → Streamlit 可视化界面
```

### 阶段 5：核心代码片段参考

**LLM 调用示例（`app/services/llm.py`）：**

```python
from openai import OpenAI

client = OpenAI(api_key="sk-...", base_url="https://api.openai.com/v1")

def generate_script(video_subject: str) -> str:
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": "你是一个短视频脚本写手。"},
            {"role": "user", "content": f"为主题'{video_subject}'写一段30秒的短视频文案。"}
        ]
    )
    return response.choices[0].message.content
```

**视频合成示例（`app/services/video.py`）：**

```python
from moviepy import (
    VideoFileClip, AudioFileClip, 
    CompositeVideoClip, TextClip, 
    CompositeAudioClip
)

def combine_video(video_clips, audio_path, subtitle_path):
    """将视频片段、音频、字幕合成一个完整视频"""
    
    # 拼接视频片段
    final_video = concatenate_videoclips(video_clips)
    
    # 添加音频
    audio = AudioFileClip(audio_path)
    final_video = final_video.with_audio(audio)
    
    # 添加字幕（简化示例）
    # ...
    
    return final_video
```

**FastAPI 路由示例（`app/controllers/v1/video.py`）：**

```python
from fastapi import APIRouter
from app.services import task as tm
from app.utils import utils

router = APIRouter()

@router.post("/videos")
def create_video(body: dict):
    task_id = utils.get_uuid()
    # 在后台启动视频生成任务
    tm.start(task_id, body)
    return {"task_id": task_id}
```

---

## 九、项目亮点与设计模式

### 1. 多 LLM 提供商支持

项目通过 `config.toml` 中的 `llm_provider` 切换不同的 AI 大模型：
- OpenAI、Moonshot、DeepSeek、通义千问、Gemini、Ollama（本地）等
- 所有提供商都通过 OpenAI 兼容接口封装，切换非常方便

### 2. 任务队列管理

支持两种任务管理器：
- **InMemoryTaskManager**：内存队列，适合单机使用
- **RedisTaskManager**：Redis 队列，适合多机分布式部署

任务状态流转：`pending → queued → processing → completed / failed`

### 3. 文件路径安全

`file_security.py` 实现了路径穿越攻击防护，确保用户不能通过 `../../` 等方式访问系统任意文件。

### 4. 多语言支持

- WebUI 支持国际化（`webui/i18n/`）
- 视频文案支持中文和英文
- 界面语言自动检测系统语言

---

## 十、常见问题

### Q: 需要什么 API Key？
- **最少需要**：一个 LLM Provider 的 API Key（如 OpenAI）
- **推荐配置**：Pexels API Key（免费注册）
- **可选**：Azure Speech Key（更高质量 TTS）

### Q: 为什么需要 Pexels API Key？
视频素材从 Pexels 下载，需要 API Key。免费注册地址：https://www.pexels.com/api/

### Q: 可以离线使用吗？
不能完全离线。至少需要 LLM API（生成文案）和 TTS 服务（生成配音）。如果使用本地素材（`video_source = "local"`）和本地 Ollama，可以减少对外部服务的依赖。

### Q: 如何扩展支持新的 LLM 提供商？
在 `config.example.toml` 中添加配置项，在 `llm.py` 中添加对应的客户端初始化逻辑即可。

---

## 十一、学习资源推荐

- **FastAPI 官方教程**：https://fastapi.tiangolo.com/zh/tutorial/
- **Streamlit 入门**：https://docs.streamlit.io/library/get-started
- **MoviePy 文档**：https://zulko.github.io/moviepy/
- **Python 官方教程**：https://docs.python.org/zh-cn/3/tutorial/
- **uv 包管理器**：https://docs.astral.sh/uv/

---

*本文档由 Claude Code 辅助生成，如有错误欢迎指正。*
