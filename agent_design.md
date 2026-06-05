# Agent 化改造方案：将固定流水线变为 LLM 动态调度

> 本文档设计如何将 `app/services/task.py` 中的固定流水线改造为 LLM 驱动的 Agent 架构。
> 适用对象：有 Python 基础的开发者 | 2026-06-04

---

## 一、当前架构的问题

### 现状：固定流水线

```
start() 函数中的硬编码顺序（task.py:250-387）:
 
 ① generate_script()    ──→  LLM 写文案
 ② generate_terms()     ──→  LLM 提取关键词
 ③ generate_audio()    ──→  TTS 配音
 ④ generate_subtitle() ──→  生成字幕
 ⑤ get_video_materials()──→  下载素材
 ⑥ generate_final_videos()─→  MoviePy 合成
                                 ↑
                    全部是硬编码 if-else，LLM 不参与决策
```

### 具体痛点

| 问题 | 表现 |
|------|------|
| **无容错弹性** | 任何一步失败，整个任务就失败。LLM 不会根据失败原因调整策略重试 |
| **无质量反馈循环** | 不会评估文案质量再决定是否进入下一步，也不会有"素材不够好→回头改关键词"的迭代 |
| **无法处理不确定性** | 如果用户说"做一个酷一点的视频"，LLM 不理解"酷"的含义，因为没有让 LLM 参与决策的机制 |
| **工具调用固定** | 永远是 Pexels → Edge TTS → MoviePy。不会根据情况选择不同工具（比如素材不够时用 Pixabay 补充） |
| **单一大模型调用** | 只在 3 个点调用 LLM，且是简单的"输入→输出"，没有让 LLM 做推理、规划、评估 |

---

## 二、改造目标：Agent 架构

### 核心思想

**让 LLM 成为"大脑"，Service 层成为"四肢"**。

LLM 不再是只用来写文案的文本生成器，而是**整个视频生成任务的规划者和调度者**。它决定：
1. **做什么** — 分析用户需求，拆解为子任务
2. **按什么顺序做** — 动态编排步骤，不一定按固定顺序
3. **用什么工具做** — 根据情况选择合适的 Service
4. **做得怎么样** — 评估中间结果，决定是否需要重做

### 架构对比

```
改造前（Orchestration）：                     改造后（Agentic）:
                                                 
  task.py（硬编码调度）                     AgentOrchestrator（LLM 驱动）
       │                                              │
       ├─→ generate_script()                 LLM 思考: "先写文案"
       ├─→ generate_terms()                       │
       ├─→ generate_audio()                  ┌─────┴──────┐
       ├─→ generate_subtitle()               │   Tool 层    │
       ├─→ get_video_materials()             │             │
       └─→ generate_final_videos()           ├─ llm.write_script()
                                              ├─ llm.extract_terms()
       LLM 只被动调用                        ├─ voice.synthesize()
                                              ├─ material.search()
                                              ├─ subtitle.create()
                                              └─ video.compose()
                                                      │
                                               LLM 思考: "素材够了吗？
                                               不够，换关键词再搜"
```

---

## 三、整体架构设计

### 3.1 新增模块

```
app/
├── agent/                          # 新增：Agent 核心
│   ├── __init__.py
│   ├── orchestrator.py             # Agent 主循环（ReAct 模式）
│   ├── tools.py                    # 工具注册表（Service 包装为 Tool）
│   ├── prompts.py                  # Agent 系统提示词
│   ├── memory.py                   # 短期记忆（当前任务上下文）
│   └── schemas.py                  # Agent 内部数据结构
│
├── services/                       # 现有代码
│   ├── task.py                     # [保留] 兼容旧的固定流水线
│   ├── llm.py                      # [保留] 现有 LLM 文本生成
│   ├── video.py                    # [保留] 视频合成
│   ├── voice.py                    # [保留] TTS 语音合成
│   ├── material.py                 # [保留] 素材下载
│   └── subtitle.py                 # [保留] 字幕生成
│
└── controllers/
    └── v1/
        └── video.py                # [修改] 增加 agent 模式的路由
```

### 3.2 ReAct 循环架构

Agent 的核心是 **ReAct（Reasoning + Acting）** 循环：

```
用户输入："做一个关于春天的视频"
         │
         ▼
┌──────────────────────────────────┐
│  ① 思考 (Think)                  │
│  LLM 分析当前状态，决定下一步动作 │
│  "用户想要春天的视频，先写文案"    │
└──────────┬───────────────────────┘
           │
           ▼
┌──────────────────────────────────┐
│  ② 行动 (Act)                    │
│  调用 Tool（包装过的 Service）    │
│  write_script(subject="春天")     │
└──────────┬───────────────────────┘
           │
           ▼
┌──────────────────────────────────┐
│  ③ 观察 (Observe)                │
│  获取 Tool 执行结果               │
│  "文案已生成，内容尚可"            │
└──────────┬───────────────────────┘
           │
           ▼
      ┌────────┐
      │ 完成?  │───否──→ 回到 ①
      └───┬────┘
          │ 是
          ▼
      返回最终结果
```

---

## 四、详细设计

### 4.1 Tool 定义（`agent/tools.py`）

将现有的 Service 函数封装成统一的 Tool 接口，供 LLM 调用：

```python
# agent/tools.py

from typing import Any, Dict, List, Optional
from pydantic import BaseModel, Field


class ToolParameter(BaseModel):
    """工具参数定义"""
    name: str
    type: str  # string, integer, boolean, etc.
    description: str
    required: bool = True
    enum: Optional[List[str]] = None  # 可选值列表


class Tool(BaseModel):
    """工具定义——LLM 通过 JSON function calling 调用的单位"""
    name: str
    description: str  # LLM 理解这个工具干什么用的
    parameters: List[ToolParameter]
    categories: List[str] = []  # "generation", "retrieval", "editing", "evaluation"

    async def execute(self, **kwargs) -> Any:
        """子类实现具体逻辑"""
        raise NotImplementedError


# ─── 具体工具实现 ───

class WriteScriptTool(Tool):
    """写视频文案"""
    name = "write_script"
    description = "根据视频主题生成视频文案/脚本"
    parameters = [
        ToolParameter(name="subject", type="string", description="视频主题"),
        ToolParameter(name="language", type="string", description="语言", required=False),
        ToolParameter(name="paragraphs", type="integer", description="段落数", required=False),
    ]
    categories = ["generation"]

    async def execute(self, **kwargs) -> str:
        # 调用现有的 llm.generate_script()
        from app.services.llm import generate_script
        return generate_script(
            video_subject=kwargs["subject"],
            language=kwargs.get("language", ""),
            paragraph_number=kwargs.get("paragraphs", 1),
        )


class ExtractTermsTool(Tool):
    """提取搜索关键词"""
    name = "extract_terms"
    description = "从视频文案中提取用于搜索视频素材的关键词"
    parameters = [
        ToolParameter(name="script", type="string", description="视频文案"),
        ToolParameter(name="subject", type="string", description="视频主题"),
    ]
    categories = ["generation"]

    async def execute(self, **kwargs) -> List[str]:
        from app.services.llm import generate_terms
        return generate_terms(
            video_subject=kwargs["subject"],
            video_script=kwargs["script"],
        )


class SynthesizeSpeechTool(Tool):
    """TTS 语音合成"""
    name = "synthesize_speech"
    description = "将文案合成为配音音频"
    parameters = [
        ToolParameter(name="script", type="string", description="要朗读的文案"),
        ToolParameter(name="voice", type="string", description="语音名称", required=False),
        ToolParameter(name="rate", type="number", description="语速", required=False),
    ]
    categories = ["generation"]

    async def execute(self, **kwargs) -> Dict:
        # 调用 voice.tts()
        ...


class SearchVideoMaterialTool(Tool):
    """搜索视频素材"""
    name = "search_video_material"
    description = "根据关键词搜索视频素材（来自 Pexels 或 Pixabay）"
    parameters = [
        ToolParameter(name="keywords", type="array", description="搜索关键词列表"),
        ToolParameter(name="source", type="string", enum=["pexels", "pixabay"], required=False),
        ToolParameter(name="aspect_ratio", type="string", enum=["16:9", "9:16", "1:1"], required=False),
    ]
    categories = ["retrieval"]

    async def execute(self, **kwargs) -> List[str]:
        ...


class EvaluateScriptQualityTool(Tool):
    """评估文案质量"""
    name = "evaluate_script_quality"
    description = "评估视频文案是否符合短视频风格、是否有吸引力"
    parameters = [
        ToolParameter(name="script", type="string", description="要评估的文案"),
        ToolParameter(name="subject", type="string", description="视频主题"),
    ]
    categories = ["evaluation"]

    async def execute(self, **kwargs) -> Dict:
        """让 LLM 自己评估文案质量"""
        prompt = f"""
        评估以下视频文案的质量，从三个维度打分（1-10分）：
        1. 吸引力：开头是否能抓住观众
        2. 流畅度：语言是否自然流畅
        3. 完整性：信息是否完整、有结尾

        视频主题：{kwargs['subject']}
        文案：{kwargs['script']}

        请以 JSON 格式返回评分和改进建议。
        """
        from app.services.llm import _generate_response
        result = _generate_response(prompt)
        # 解析结果并返回结构化数据
        return {"score": ..., "suggestions": ...}


class ComposeVideoTool(Tool):
    """合成最终视频"""
    name = "compose_video"
    description = "将素材视频、音频、字幕合成为最终视频"
    parameters = [...]
    categories = ["editing"]

    async def execute(self, **kwargs) -> str:
        ...


# ─── 工具注册表 ───

TOOL_REGISTRY: Dict[str, Tool] = {
    "write_script": WriteScriptTool(),
    "extract_terms": ExtractTermsTool(),
    "synthesize_speech": SynthesizeSpeechTool(),
    "search_video_material": SearchVideoMaterialTool(),
    "evaluate_script_quality": EvaluateScriptQualityTool(),
    "compose_video": ComposeVideoTool(),
    # ... 更多工具
}
```

### 4.2 Agent 主循环（`agent/orchestrator.py`）

```python
# agent/orchestrator.py

"""
ReAct Agent 主循环

LLM 在每一步决定:
  1. 当前应该调用哪个 Tool（或者任务已完成）
  2. 传入什么参数
  3. 收到结果后，判断下一步做什么

直到 LLM 输出 "task_complete" 信号。
"""

import json
from typing import Any, Dict, List, Optional
from loguru import logger

from app.agent.tools import TOOL_REGISTRY, Tool
from app.agent.memory import TaskMemory
from app.agent.prompts import SYSTEM_PROMPT, build_step_prompt


class AgentOrchestrator:
    """
    Agent 编排器——运行 ReAct 循环直到任务完成。
    """

    def __init__(self, task_id: str, user_params: Dict[str, Any]):
        self.task_id = task_id
        self.params = user_params
        self.memory = TaskMemory(task_id)  # 短期记忆
        self.max_iterations = 20           # 防止无限循环
        self.iteration = 0

    async def run(self) -> Dict[str, Any]:
        """
        主循环：Think → Act → Observe，直到完成。
        """
        # 初始状态：记录用户需求
        self.memory.add_event("user_input", self.params)

        while self.iteration < self.max_iterations:
            self.iteration += 1
            logger.info(f"Agent iteration {self.iteration}")

            # ── Step 1: Think ──
            # LLM 根据当前记忆决定下一步
            next_action = await self._think()
            
            if next_action["type"] == "task_complete":
                logger.success(f"Agent decided task complete: {next_action.get('reason', '')}")
                break

            if next_action["type"] == "error":
                logger.error(f"Agent encountered error: {next_action.get('message', '')}")
                # 可以让 LLM 决定是否重试或放弃
                ...

            # ── Step 2: Act ──
            tool_result = await self._act(next_action)

            # ── Step 3: Observe ──
            self.memory.add_event("observation", {
                "action": next_action,
                "result": tool_result,
            })

            # 更新外部任务状态
            from app.services.state import state
            state.update_task(self.task_id, progress=min(95, self.iteration * 10))

        # 返回最终结果
        return self.memory.get_final_result()

    async def _think(self) -> Dict:
        """
        让 LLM 思考下一步行动。

        LLM 返回结构化 JSON:
          {"type": "call_tool", "tool": "write_script", "args": {...}}
          {"type": "task_complete", "reason": "所有步骤已完成"}
          {"type": "error", "message": "无法继续"}
        """
        prompt = build_step_prompt(
            system=SYSTEM_PROMPT,
            tools=[t.model_dump() for t in TOOL_REGISTRY.values()],
            memory=self.memory.get_history(),
        )

        # 调用 LLM（复用现有的 _generate_response，但改为结构化输出）
        from app.services.llm import _generate_response
        response = _generate_response(prompt)

        # 解析 LLM 返回的 JSON 决策
        try:
            return json.loads(response)
        except json.JSONDecodeError:
            logger.warning(f"LLM returned invalid JSON: {response}")
            return {"type": "error", "message": "LLM output parsing failed"}

    async def _act(self, decision: Dict) -> Any:
        """
        根据 LLM 的决策调用对应的 Tool。
        """
        tool_name = decision.get("tool")
        args = decision.get("args", {})

        if tool_name not in TOOL_REGISTRY:
            return {"error": f"Unknown tool: {tool_name}"}

        tool = TOOL_REGISTRY[tool_name]
        logger.info(f"Executing tool: {tool_name}({args})")

        try:
            result = await tool.execute(**args)
            self.memory.add_event("tool_result", {
                "tool": tool_name,
                "result": result,
            })
            return result
        except Exception as e:
            logger.error(f"Tool {tool_name} failed: {e}")
            return {"error": str(e)}
```

### 4.3 系统提示词（`agent/prompts.py`）

```python
# agent/prompts.py

SYSTEM_PROMPT = """
你是一个 AI 短视频生成助手。你的任务是根据用户的输入，生成一个高质量的短视频。

## 工作方式
你会通过"思考→行动→观察"循环逐步完成工作。每次循环：
1. 思考当前状态，决定下一步行动
2. 调用一个工具来执行
3. 观察工具返回的结果
4. 重复直到任务完成

## 可用工具
{tools_description}

## 工具选择指南
- 先写文案，再提取关键词
- 如果文案质量不够好（吸引力<7分），应该重写
- 搜索素材时可以尝试多个关键词组合
- 如果某个关键词没有搜到素材，换一个关键词试试
- 合成视频需要依赖：文案 + 音频 + 素材 + 字幕（可选）

## 规则
1. 一次只调用一个工具
2. 每次调用后观察结果再决定下一步
3. 如果某个步骤失败（error），尝试换一种方式重试
4. 最多重试 3 次
5. 全部完成后输出：{{"type": "task_complete", "reason": "..."}}
6. 如果遇到无法解决的问题，输出：{{"type": "error", "message": "..."}}

## 输出格式
你的输出必须是以下三种格式之一：
1. 调用工具：{{"type": "call_tool", "tool": "工具名", "args": {{...}}}}
2. 完成：  {{"type": "task_complete", "reason": "..."}}
3. 错误：  {{"type": "error", "message": "..."}}
"""


def build_step_prompt(
    system: str,
    tools: List[Dict],
    memory: List[Dict],
) -> str:
    """构建当前步骤的完整提示词"""

    # 工具描述
    tools_desc = "\n".join([
        f"- {t['name']}: {t['description']}"
        f"  参数: {[p['name'] for p in t['parameters']]}"
        for t in tools
    ])

    # 当前任务记忆
    memory_desc = "\n".join([
        f"[{m['role']}] {m['content']}"
        for m in memory[-10:]  # 只保留最近 10 条，避免上下文过长
    ])

    prompt = f"""{system.format(tools_description=tools_desc)}

## 当前任务上下文
{memory_desc}

## 接下来你决定做什么？
"""
    return prompt
```

### 4.4 短期记忆（`agent/memory.py`）

```python
# agent/memory.py

from typing import Any, Dict, List


class TaskMemory:
    """
    任务的短期记忆。

    记录整个 Agent 运行过程中的所有事件，包括：
    - 用户输入
    - LLM 决策
    - 工具调用及结果
    - 中间产物路径
    - 质量评估分数
    """

    def __init__(self, task_id: str):
        self.task_id = task_id
        self.events: List[Dict] = []
        self.artifacts: Dict[str, Any] = {}  # 中间产物

    def add_event(self, role: str, content: Any):
        """记录一个事件"""
        self.events.append({
            "role": role,        # user_input, llm_decision, tool_result, observation
            "content": content,
        })

    def get_history(self) -> List[Dict]:
        """获取完整历史"""
        return self.events

    def set_artifact(self, key: str, value: Any):
        """保存中间产物（文案、音频路径等）"""
        self.artifacts[key] = value

    def get_artifact(self, key: str) -> Any:
        return self.artifacts.get(key)

    def get_final_result(self) -> Dict:
        """组装最终返回结果"""
        return {
            "task_id": self.task_id,
            "artifacts": self.artifacts,
            "events_count": len(self.events),
        }
```

### 4.5 API 路由改造

在现有的 `app/controllers/v1/video.py` 中增加一个 Agent 模式接口：

```python
# app/controllers/v1/video.py (新增路由)

from app.agent.orchestrator import AgentOrchestrator

@router.post("/agent/videos", summary="[Agent模式] 智能生成视频")
async def create_video_agent(request: Request, body: TaskVideoRequest):
    """
    使用 Agent 模式生成视频。
    LLM 会自主规划步骤、评估质量、动态调整。
    """
    task_id = utils.get_uuid()
    
    # 初始化 Agent
    agent = AgentOrchestrator(
        task_id=task_id,
        user_params=body.model_dump(),
    )
    
    # 在后台运行 Agent
    background_tasks.add_task(agent.run)
    
    return utils.get_response(200, {"task_id": task_id})
```

---

## 五、运行示例

### 场景：用户说"做一个关于春天的视频"

以下是一个完整的 Agent 运行轨迹：

```
Iteration 1:
  Think:    "用户想要关于春天的视频，先写文案"
  Act:      write_script(subject="春天", paragraphs=2)
  Observe:  "春天的花海，是大自然的一幅美丽画卷..."

Iteration 2:
  Think:    "文案写好了，先评估一下质量再继续"
  Act:      evaluate_script_quality(script="春天的花海...")
  Observe:  {"score": 8, "suggestions": "开头可以更吸引人"}

Iteration 3:
  Think:    "质量还可以（8分），继续下一步。提取搜索关键词"
  Act:      extract_terms(script="春天的花海...", subject="春天")
  Observe:  ["spring flowers", "blooming garden", "butterfly", ...]

Iteration 4:
  Think:    "先搜第一组关键词看看素材质量"
  Act:      search_video_material(keywords=["spring flowers"], source="pexels")
  Observe:  ["/storage/cache_videos/spring1.mp4", ...]  (搜到5个素材)

Iteration 5:
  Think:    "只有5个素材，可能不够。再换一组关键词补充"
  Act:      search_video_material(keywords=["blooming garden"], source="pexels")
  Observe:  ["/storage/cache_videos/garden1.mp4", ...]  (再搜到3个)

Iteration 6:
  Think:    "素材够了（8个片段）。现在生成配音"
  Act:      synthesize_speech(script="春天的花海...", voice="zh-CN-Xiaoxiao")
  Observe:  {"audio_file": "/storage/tasks/xxx/audio.mp3", "duration": 30}

Iteration 7:
  Think:    "配音时长30秒，素材总共8个片段，每个约5秒，足够拼接。
            开始合成最终视频"
  Act:      compose_video(materials=[...], audio="...", subtitle=True)
  Observe:  {"video": "/storage/tasks/xxx/final-1.mp4"}

Iteration 8:
  Think:    "视频已生成，所有步骤完成"
  Complete: {"type": "task_complete", "reason": "视频已成功生成"}
```

### 对比：固定流水线遇到问题时的处理

| 场景 | 固定流水线 | Agent 模式 |
|------|-----------|-----------|
| 素材不够 | 直接失败 | 换关键词再搜，或者从 Pixabay 补充 |
| 文案质量差 | 不评估，直接往下走 | 评分 < 7 分时让 LLM 重写 |
| 配音太短/太长 | 不检查 | 检查时长，不够则调整文案或语速 |
| 用户需求模糊 | 无法处理 | LLM 可主动提问澄清 |

---

## 六、实施计划

### 阶段 1：基础设施（1-2 天）

```
□ 创建 app/agent/ 目录结构
□ 实现 agent/tools.py — Tool 基类 + 注册表
□ 实现 agent/prompts.py — 系统提示词
□ 实现 agent/memory.py — 短期记忆
□ 实现 agent/schemas.py — Agent 内部数据结构
```

### 阶段 2：核心循环（2-3 天）

```
□ 实现 agent/orchestrator.py — ReAct 主循环
□ 对接现有的 _generate_response() 作为 LLM 调用后端
□ 实现 LLM 输出解析（JSON 提取与校验）
□ 添加最大迭代限制和超时保护
```

### 阶段 3：工具实现（2-3 天）

```
□ 包装现有的 Service 为 Tool（逐个实现）
  □ WriteScriptTool
  □ ExtractTermsTool
  □ EvaluateScriptQualityTool  ← 新增，LLM 自我评估
  □ SynthesizeSpeechTool
  □ SearchVideoMaterialTool
  □ ComposeVideoTool
□ 为每个 Tool 添加重试和错误处理逻辑
```

### 阶段 4：集成与 API（1-2 天）

```
□ 在 video.py 添加 /agent/videos 路由
□ 兼容现有任务状态管理（state.py）
□ 添加 Agent 运行日志
□ 确保 /tasks/{id} 查询接口同时支持 Agent 模式任务
```

### 阶段 5：质量提升（持续）

```
□ 优化系统提示词，让 LLM 更好地理解"什么样的视频是好视频"
□ 添加更细粒度的评估工具（画面质量、配音自然度等）
□ 支持用户中途干预（暂停 Agent、修改参数后继续）
□ 支持记忆持久化，让 Agent 记住用户偏好
```

---

## 七、风险与注意事项

### 1. LLM 输出不稳定
LLM 可能返回非 JSON 格式或不合理的决策。必须：
- 健壮的 JSON 解析（包含正则 fallback）
- 参数 schema 校验
- 重复相同决策时的去重保护

### 2. Token 消耗显著增加
```
固定流水线: 2-3 次 LLM 调用
Agent 模式: 可能 10-20 次 LLM 调用（每次决策都调用）
```
解决方案：使用便宜的推理模型（如 GPT-4o-mini、DeepSeek）作为"规划大脑"，
只在需要高质量文本生成时切换到更强的模型。

### 3. 无限循环风险
```
□ max_iterations = 20 硬限制
□ 检测重复循环（连续 3 次相同决策 → 强制终止）
□ 超时保护（单次 Agent 运行最多 10 分钟）
```

### 4. 调试难度增加
固定流水线的问题是确定的，Agent 的行为具有随机性。建议：
```
□ 完整记录 LLM 的每次思考和决策（think log）
□ 支持回放模式（replay mode）：复用之前的决策序列，只重放执行
□ 开发时可用固定 seed 让 LLM 输出更稳定
```

---

## 八、快速原型：最小可验证版本

如果你只想先试试 Agent 模式的效果，可以先实现一个"最小版本"：

### 最小 Agent（约 100 行）

```python
# app/agent/mini_agent.py

"""
最小可验证 Agent——在现有的固定流水线基础上，
只在关键决策点加入 LLM 评估和分支。
"""

import json
from loguru import logger

from app.services import llm, voice, material, video, subtitle
from app.services import state as sm
from app.models import const
from app.utils import utils


def run_agent(task_id, params):
    """
    轻量级 Agent：
    固定流水线 + LLM 在关键节点的评估和决策。
    """
    
    # 1. 生成文案 + LLM 自我评估
    script = llm.generate_script(params.video_subject, ...)
    
    # ★ Agent 改进：评估文案质量，不合格就重写
    for retry in range(2):
        quality = llm._generate_response(f"""评估这段文案的质量（1-10分），只返回分数：
        {script}""")
        score = int(re.search(r"\d+", quality).group())
        if score >= 7:
            break
        logger.warning(f"脚本评分{score}，低于7分，重新生成...")
        script = llm.generate_script(
            params.video_subject,
            video_script_prompt=f"请写得更好。上版评分{score}/10。改进建议：{quality}"
        )
    
    # 2-4. 原样保留 generate_terms → generate_audio → generate_subtitle
    ...
    
    # 5. 获取素材
    terms = ...
    videos = material.download_videos(...)
    
    # ★ Agent 改进：素材不够时自动补搜
    if len(videos) < 3:
        logger.warning(f"素材不足({len(videos)}个)，补充搜索")
        more_terms = llm.generate_terms(
            params.video_subject, script, amount=3
        )
        more_videos = material.download_videos(..., search_terms=more_terms)
        videos.extend(more_videos)
    
    # 6. 合成视频
    ...
```

这个最小版本只需要修改 `task.py` 中的 `start()` 函数，在 2-3 个关键节点插入 LLM 评估逻辑，就能快速体验 Agent 模式的效果。

---

## 九、总结

| 维度 | 固定流水线（现状） | Agent 模式（目标） |
|------|------------------|------------------|
| LLM 角色 | 文本生成器 | 规划者 + 决策者 + 评估者 |
| 流程控制 | 硬编码 if-else | LLM 动态决策 |
| 容错能力 | 一步失败全员失败 | 自动重试、换方案 |
| 质量反馈 | 无 | LLM 自评估循环 |
| 复杂度 | 低 | 中高 |
| Token 消耗 | 低 | 中高 |
| 结果质量 | 稳定但上限低 | 波动但上限高 |

**建议实施策略**：
1. 先用 **最小 Agent**（第八章）在现有代码上快速验证效果
2. 验证通过后再完整实现 **ReAct 架构**
3. 保留两种模式并存（用户可选 `mode="pipeline"` 或 `mode="agent"`）

---

*本文档由 Claude Code 辅助生成*
