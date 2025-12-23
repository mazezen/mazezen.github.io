---
layout: post
title: 使用OpenAI结合SerpApi构建一个简易的 AI Agent(AI智能体)
date: 2025-12-23
tags: [AI]
comments: true
author: mazezen
---

# 使用 OpenAI 大模型 结合 SerpApi 构建一个简易的 AI Agent(AI 智能体)

<a href="https://github.com/mazezen/hello-agents/tree/master">查看完整代码</a>

> 最近在读<<从零开始构建智能体>>这本书,学习研究 AI Agent.在此记录本次的学习成功及流程.
>
> 本篇文章大部分代码来自于 <<从零开始构建智能体>>.与书中不同的是,书中采用的是 Qwen 大模型. 而我采用的是 ChatGPT 大模型. 大同小异
> 完整代码请查看 <a href="https://github.com/mazezen/hello-agents/tree/master/04build-agent">build-agent</a>

## Ai Agent 简介

> AI 代理（AI Agent）是指一种基于人工智能技术构建的自动化系统或软件，它能够执行特定任务、做出决策并与外部环境或用户进行交互。AI 代理通常具备感知、推理、规划、学习和行动等功能，使其能够模拟人的行为或表现出智能行为。

## AI Agent 的特征：

1. 感知（Perception）：
   AI 代理通过传感器（如文本输入、视觉输入等）感知环境或接收到的信息。例如，它可以通过处理自然语言文本来理解用户的需求，或者通过传感器获取现实世界的数据。

2. 推理与决策（Reasoning & Decision Making）：
   基于感知的数据，AI 代理会运用推理模型、规则或算法来分析当前的情境，并做出合理的决策。推理可能包括逻辑推理、概率推理或深度学习推理等。

3. 行动（Action）：
   在做出决策后，AI 代理会执行适当的行动。例如，它可能调用外部工具（如搜索引擎、数据库等），或者与用户互动（如发送回复、执行指令等）。

4. 学习（Learning）：
   高级的 AI 代理可以从环境中学习，并随着时间的推移优化其行为。这种能力常见于基于机器学习（尤其是深度学习）构建的代理系统。通过反馈循环，AI 代理能够不断改进其决策过程和任务执行。

5. 自我管理（Autonomy）：
   AI 代理通常具备一定的自主性，它可以独立执行任务，而不需要持续的人工干预。例如，在无人驾驶汽车中，AI 代理需要独立决策和行动来控制车辆。

6. 协作与交互（Collaboration & Interaction）：
   AI 代理不仅可以单独工作，还可以与其他代理或人类进行协作。例如，多智能体系统中，多个 AI 代理协同工作来完成更复杂的任务。

## 如何构建一个 Agent?

> 需要用到的工具及包有:
>
> python >= 3.10.0
>
> pip install openai python-dotenv google-search-results
>
> openai: python 语言封装对接 LLM 的包
>
> python-dotenv: 读取 .env 配置文件的包
>
> google-search-results: 用于获取 Google 搜索结果, 而不需要再对网页进行解析

### 1. 创建 env 文件.配置需要用的 API KEY

> 注册 openai 账号获取, API KEY.
>
> 注册 Serp API 账号获取 API KEY.

```env
LLM_API_KEY=""
LLM_MODEL_ID="gpt-5-nano"
LLM_BASE_URL="https://api.openai.com/v1"
LLM_TIMEOUT=60
SERPAPI_API_KEY=""
```

### 2. 编写 tools 工具,用户通过 Google 搜索结果. 供大模型使用.

```python
def search(query: str) -> str:
    """
    一个基于SerApi的实战网页搜索工具
    他会智能地解析搜索结果,优先返回直接答案或指示图谱信息
    """
    print(f"🔍 正在执行 [SerApi] 网页搜索: {query}")
    try:
        api_key = os.getenv("SERPAPI_API_KEY")
        if not api_key:
            return "错误: SERPAPI_API_KEY 未在 .env文件中配置."

        params = {
            "engine": "google",
            "q": query,
            "api_key": api_key,
            "gl": "cn", # 国家代码
            "hl": "zh-cn", # 语言代码
        }

        client = SerpApiClient(params)
        results = client.get_dict()

        if "answer_box_list" in results:
            return "\n".join(results["answer_box_list"])
        if "answer_box" in results and "answer" in results["answer_box"]:
            return results["answer_box"]["answer"]
        if "knowledge_graph" in results and "description" in results["knowledge_graph"]:
            return results["knowledge_graph"]["description"]
        if "organic_results" in results and results["organic_results"]:
            snippets = [
                f"[{i+1}] {res.get('title', '')}\n{res.get('snippet', '')}"
                for i, res in enumerate(results["organic_results"])
            ]
            return "\n\n".join(snippets)

        return f"对不起, 没有找到关于 '{query}'的信息."
    except Exception as e:
        return f"搜索时发生错误: {e}"
```

### 3. 创建 LLM 大型客户端

```python
class HelloAgentsLLM:
    """
    它用于调用任何兼容OpenAI接口的服务, 并默认使用流式响应
    """
    def __init__(self, model: str = None, apikey: str = None, baseUrl: str = None, timeout: int = None):
        """
        初始化客户端。优先使用传入参数，如果未提供，则从环境变量加载
        """
        self.model = model or os.getenv("LLM_MODEL_ID")
        apikey = apikey or os.getenv("LLM_API_KEY")
        baseUrl = baseUrl or os.getenv("LLM_BASE_URL")
        timeout = timeout or int(os.getenv("LLM_TIMEOUT", 60))

        if not all([self.model, apikey, baseUrl]):
            raise ValueError("模型ID, API 秘钥和服务地址必须被提供或者在.env文件中定义.")

        self.client = OpenAI(api_key=apikey, base_url=baseUrl, timeout=timeout)

    def think(self, messages: List[Dict[str, str]], temperature: float = 0) -> str:
        """
        调用大语言模型进行思考，并返回其响应。

        :param self: 说明
        :param messages: 说明
        :type messages: List[Dict[str, str]]
        :param temperature: 说明
        :type temperature: float
        :return: 说明
        :rtype: str
        """
        print(f"🧠 正在调用 {self.model} 模型...")
        try:
            response = self.client.chat.completions.create(
                model=self.model,
                messages=messages,
                # temperature=temperature,
                stream=True,
            )
            print("✅ 大语言模型响应成功:")
            collected_content = []
            for chunk in response:
                content = chunk.choices[0].delta.content or ""
                print(content, end="", flush=True)
                collected_content.append(content)
            print()
            return "".join(collected_content)
        except Exception as e:
            print(f"❌ 调用LLM API时发生错误: {e}")
            return None
```

### 4. 编写 Prompt,告诉大模型角色定位,执行流程,可用的工具等

```text
# ReAct 提示词模版
REACT_PROMPT_TEMPLATE = """
请注意, 你是一个有能力调用外部工具的智能助手.

可用工具如下:
{tools}

请严格按照以下合适进行回应:

Thought: 你的思考过程，用于分析问题、拆解任务和规划下一步行动。
Action: 你决定采取的行动，必须是以下格式之一:
- `{{tool_name}}[{{tool_input}}]`:调用一个可用工具。
- `Finish[最终答案]`:当你认为已经获得最终答案时。
- 当你收集到足够的信息,能够回答用户的最终问题时,你必须在Action:字段后使用 finish(answer="...") 来
输出最终答案。

现在,请开始解决以下问题:
Question: {question}
History: {history}
"""
```

### 5. 智能体的入口,调用大模型客户端,及工具清单

```python
def run(self, question: str):
    self.history = []
    current_step = 0

    while current_step < self.max_steps:
        current_step += 1
        print(f"--- 第 {current_step} 步 ---")

        tools_desc = self.tool_exectuor.getAvailableTools()
        history_str = "\n".join(self.history)
        prompt = REACT_PROMPT_TEMPLATE.format(
            tools=tools_desc,
            question=question,
            history=history_str
        )

        messages = [{"role": "user", "content": prompt}]
        response_text = self.llm_client.think(messages=messages)

        if not response_text:
            print("错误:LLM未能返回有效回应")
            break

        thought, action = self._parse_output(response_text)

        if thought:
            print(f"🤔 思考: {thought}")

        if not action:
            print("⚠️ 警告:未能解析出有效的Action,流程终止.")
            break

        if action.startswith("Finish"):
            final_answer = re.match(r"Finish\[(.*)\]", action).group(1)
            print(f"🎉 最终答案: {final_answer}")
            return final_answer

        tool_name, tool_input = self._parse_action(action)
        if not tool_name or not tool_input:
            continue

        print(f"🎬 行动: {tool_name}[{tool_input}]")

        too_function = self.tool_exectuor.getTool(tool_name)
        if not too_function:
            observation = f"❌ 错误:未找到名为 '{tool_name}' 的工具"
        else:
            observation = too_function(tool_input)

        print(f"👀 观察: {observation}")

        self.history.append(f"Action: {action}")
        self.history.append(f"Observation: {observation}")

    print("已达到最大步数，流程终止。")
    return None
```

### 6. 解析器: 解析大模型的输出

```python
def _parse_output(self, text: str):
    thought_match = re.search(r"Thought: (.*)", text)
    action_match = re.search(r"Action: (.*)", text)
    thought = thought_match.group(1).strip() if thought_match else None
    action = action_match.group(1).strip() if action_match else None
    return thought, action

def _parse_action(self, action_text: str):
    if action_text.startswith("Finish"):
        final_answer_match = re.match(r"Finish\[answer=\"(.*)\"]", action_text)
        if final_answer_match:
            return "Finish", final_answer_match.group(1)

    match = re.match(r"(\w+)\[(.*)\]", action_text)
    if match:
        return match.group(1), match.group(2)
    return None, None
```

### 7. 运行查看结果

```shell
工具 'Search' 已注册.
--- 第 1 步 ---
🧠 正在调用 gpt-5-nano 模型...
✅ 大语言模型响应成功:
Action: Search["华为 最新 手机 2025 机型 以及 主要卖点"]

Finish[answer="我需要先检索最新信息以确保准确。请稍等，我将查证并给出当前华为的最新机型及其主要卖点。"]
🎬 行动: Search["华为 最新 手机 2025 机型 以及 主要卖点"]
🔍 正在执行 [SerApi] 网页搜索: "华为 最新 手机 2025 机型 以及 主要卖点"
👀 观察: [1] 2025年华为手机哪一款性价比高？华为手机推荐与市场分析
华为Nova系列中端机型，主打拍照能力和性价比，影像能力比上一代进步不少，机子的颜值和手感也很到位。 人像拍照效果好，且有鸿蒙系统加持，可玩性更高。 前置6000万超高像素镜 ...

[2] 2025年华为手机各系列介绍及选购指南（12月更新） ...
华为Pura系列为主打拍照的旗舰系列，nova系列主打外观及自拍的中端系列，畅享系列主打入门系列。 一、华为Mate系列（5000元以上）. 华为Mate80系列参数对比如图：. 目前更推荐 ...

[3] 华为手机- 华为官网
探索并选购华为最新手机，了解Mate 系列、Pura 系列、Pocket 系列、nova 系列、畅享系列及相关配件，体验鸿蒙AI 、影像、通信等功能。

[4] 华为nova 15系列突然官宣：三款机型齐发且卖点清晰
机身背部采用的是横向布置的双环影像模组，这种布局不同于以往nova系列的纵向排列，视觉上更显稳定与开阔。 展开剩余78 %.

[5] 华为Pura 80系列太卷了：四款机型应该怎么选
6月11日华为发布Pura 80系列手机共四款。Pura 80 Pro、Pura 80 Pro+于14日开售，售价6499元、7999元起；Pura 80 Ultra 9999元起，26日开售；标准版7月开售 ...

[6] 2025华为手机最建议买什么型号？铁粉推荐这5款，老型号上榜
这次挑了5款机型，覆盖4000-12000价位段，既有2025年新旗舰，也有“钉子户”老型号。 ... 主要配置：. 核心硬件：麒麟9020 处理器，运行HarmonyOS 5.1；16 ...

[7] 从尝鲜到实用，折叠屏手机“卷”向软件生态
近日，华为发布三折叠屏手机Mate XTs，相比上一代，该机型不仅在铰链、抗冲击性等硬件层面持续升级，还首次实现PC级多窗交互，支持原生运行PC版WPS Office、 ...

[8] 华为Pura X阔折叠发布，一文看懂华为折叠屏手机布局
华为推出的折叠屏机型种类非常多，可以看到入门级的Nova Flip小折叠，到高端的Mate XT非凡大师三折叠，再到最新发布的Pura X阔折叠。华为折叠屏在旗舰机型 ...

[9] 华为发布HUAWEI Pura 80系列，让影像远超想象
HUAWEI Pura 80 Pro+搭载1英寸高动态主摄，配合红枫原色摄像头的像素级色彩校准，明暗细节清晰，色彩还原精准。HUAWEI Pura 80 Ultra首发“聚光宝盒”1英寸超高 ...

[10] 华为手机排行榜：2025年最新顶尖手机推荐 - 三木科普
如果你追求性价比，华为Nova11绝对是2025年最值得考虑的手机之一。作为华为的中端系列，Nova11不仅具备出色的外观设计，还配备了麒麟9000E处理器，能够满足日常使用和中等强度 ...
--- 第 2 步 ---
🧠 正在调用 gpt-5-nano 模型...
✅ 大语言模型响应成功:
Action: Finish[answer="华为最新的手机是华为 Pura 80 系列（其中 Pura 80 Ultra 为本轮旗舰，另有 Pura 80 Pro、Pura 80 Pro+，以及标准版在同一发布周期内同步推进）。主要卖点包括：\n- 1英寸高动态范围主摄（Pura 80 Pro+ 与 Pura 80 Ultra 均具备）— 暗部细节丰富、动态范围更广；\n- 红枫原色摄像头与像素级色彩校准，色彩还原精准、画质稳定；\n- Pura 80 Ultra 独有的“聚光宝盒”光效提升，提升低光与夜景拍摄表现；\n- 鸿蒙 AI 与 HarmonyOS 5.x 的深度协同，摄影/视频等功能体验优化；\n- 麒麟9020 处理器与系统级优化，提供强劲性能与能效表现；\n- 发布与开售时间：Pura 80 Pro/Pro+ 14 日开售，起售价 6499 元/7999 元；Pura 80 Ultra 起售价 9999 元，26 日开售，标准版7月开售。"]
🎉 最终答案: answer="华为最新的手机是华为 Pura 80 系列（其中 Pura 80 Ultra 为本轮旗舰，另有 Pura 80 Pro、Pura 80 Pro+，以及标准版在同一发布周期内同步推进）。主要卖点包括：\n- 1英寸高动态范围主摄（Pura 80 Pro+ 与 Pura 80 Ultra 均具备）— 暗部细节丰富、动态范围更广；\n- 红枫原色摄像头与像素级色彩校准，色彩还原精准、画质稳定；\n- Pura 80 Ultra 独有的“聚光宝盒”光效提升，提升低光与夜景拍摄表现；\n- 鸿蒙 AI 与 HarmonyOS 5.x 的深度协同，摄影/视频等功能体验优化；\n- 麒麟9020 处理器与系统级优化，提供强劲性能与能效表现；\n- 发布与开售时间：Pura 80 Pro/Pro+ 14 日开售，起售价 6499 元/7999 元；Pura 80 Ultra 起售价 9999 元，26 日开售，标准版7月开售。"
```

<a href="https://github.com/mazezen/hello-agents/tree/master">查看完整代码</a>
