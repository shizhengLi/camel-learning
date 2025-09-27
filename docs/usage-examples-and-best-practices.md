# CAMEL框架使用示例和最佳实践

## 目录

1. [快速开始](#1-快速开始)
2. [基础智能体使用](#2-基础智能体使用)
3. [多智能体协作](#3-多智能体协作)
4. [工具集成](#4-工具集成)
5. [记忆系统配置](#5-记忆系统配置)
6. [模型管理优化](#6-模型管理优化)
7. [任务管理系统](#7-任务管理系统)
8. [高级应用场景](#8-高级应用场景)
9. [性能优化](#9-性能优化)
10. [错误处理和调试](#10-错误处理和调试)

---

## 1. 快速开始

### 1.1 环境配置

```bash
# 安装CAMEL框架
pip install camel-ai

# 或者从源码安装
git clone https://github.com/camel-ai/camel.git
cd camel
pip install -e .

# 配置环境变量
export OPENAI_API_KEY="your-openai-api-key"
export ANTHROPIC_API_KEY="your-anthropic-api-key"
```

### 1.2 第一个智能体

```python
from camel.agents import ChatAgent
from camel.models import ModelFactory
from camel.types import ModelPlatformType, ModelType

# 创建模型
model = ModelFactory.create(
    model_platform=ModelPlatformType.OPENAI,
    model_type=ModelType.GPT_4O_MINI,
    model_config_dict={"temperature": 0.7}
)

# 创建智能体
agent = ChatAgent(
    system_message="You are a helpful assistant.",
    model=model
)

# 进行对话
response = agent.step("Hello, how are you?")
print(response.msgs[0].content)
```

### 1.3 简单的角色扮演

```python
from camel.societies import RolePlaying

# 创建角色扮演会话
role_play = RolePlaying(
    assistant_role_name="Python Programmer",
    user_role_name="Stock Trader",
    task_prompt="Develop a trading bot for the stock market",
    with_task_specify=True,
    model=model
)

# 开始对话
input_msg = role_play.init_chat()
assistant_response, user_response = role_play.step(input_msg)

print("Assistant:", assistant_response.msgs[0].content)
print("User:", user_response.msgs[0].content)
```

---

## 2. 基础智能体使用

### 2.1 ChatAgent的多种初始化方式

```python
from camel.agents import ChatAgent
from camel.models import ModelFactory
from camel.types import ModelPlatformType, ModelType

# 方法1: 使用字符串指定模型
agent1 = ChatAgent("You are a helpful assistant.", model="gpt-4o-mini")

# 方法2: 使用枚举指定模型
agent2 = ChatAgent(
    "You are a helpful assistant.",
    model=ModelType.GPT_4O_MINI
)

# 方法3: 使用元组指定平台和模型
agent3 = ChatAgent(
    "You are a helpful assistant.",
    model=("anthropic", "claude-3-5-sonnet-latest")
)

# 方法4: 使用预创建的模型
model = ModelFactory.create(
    model_platform=ModelPlatformType.OPENAI,
    model_type=ModelType.GPT_4O_MINI
)
agent4 = ChatAgent("You are a helpful assistant.", model=model)

# 方法5: 使用模型管理器（负载均衡）
from camel.models import ModelManager

model_manager = ModelManager(
    models=["gpt-4o-mini", "claude-3-5-sonnet-latest"],
    scheduling_strategy="round_robin"
)
agent5 = ChatAgent("You are a helpful assistant.", model=model_manager)
```

### 2.2 智能体配置最佳实践

```python
from camel.memories import ChatHistoryMemory
from camel.toolkits import SearchToolkit
from camel.toolkits.function_tool import FunctionTool

# 完整的智能体配置
agent = ChatAgent(
    # 1. 系统消息定义角色
    system_message="""You are a helpful assistant specialized in data analysis and
    web research. You can search the web for current information and help users
    analyze data.""",

    # 2. 模型配置
    model=ModelFactory.create(
        model_platform="openai",
        model_type="gpt-4o-mini",
        model_config_dict={
            "temperature": 0.3,  # 降低随机性，提高一致性
            "max_tokens": 2000,  # 限制响应长度
            "top_p": 0.9         # 核采样参数
        }
    ),

    # 3. 记忆系统
    memory=ChatHistoryMemory(
        window_size=50,        # 保存最近50条消息
        token_limit=4000,      # Token限制
        enable_tool_calling_messages=False  # 清理工具调用消息
    ),

    # 4. 工具集成
    tools=[
        FunctionTool(SearchToolkit().search_brave),
        FunctionTool(SearchToolkit().search_bing)
    ],

    # 5. 响应控制
    response_terminators=[TokenLimitTerminator(2000)],

    # 6. 错误处理和重试
    retry_attempts=3,
    retry_delay=1.0,
    step_timeout=30.0,

    # 7. 输出语言
    output_language="English"
)
```

### 2.3 消息处理最佳实践

```python
from camel.messages import BaseMessage

# 创建不同类型的消息
system_msg = BaseMessage.make_system_message(
    role_name="System",
    content="You are a helpful Python programming assistant."
)

user_msg = BaseMessage.make_user_message(
    role_name="Developer",
    content="How do I create a list comprehension in Python?"
)

assistant_msg = BaseMessage.make_assistant_message(
    role_name="Python Assistant",
    content="A list comprehension in Python is a concise way to create lists..."
)

# 多模态消息（支持图像）
from PIL import Image

image = Image.open("code_example.png")
multimodal_msg = BaseMessage.make_user_message(
    role_name="Developer",
    content="What's wrong with this code?",
    image_list=[image],
    image_detail="high"
)

# 批量处理消息
messages = [system_msg, user_msg]
response = agent.step(messages)
```

---

## 3. 多智能体协作

### 3.1 基础角色扮演

```python
from camel.societies import RolePlaying

def run_role_playing_session():
    """运行完整的角色扮演会话"""

    # 1. 创建角色扮演会话
    role_play = RolePlaying(
        assistant_role_name="Python Developer",
        user_role_name="Code Reviewer",
        task_prompt="Develop and review a Python function for data processing",
        with_task_specify=True,      # 启用任务规范
        with_task_planner=True,      # 启用任务规划
        model=model
    )

    # 2. 显示任务信息
    print(f"Original task: {role_play.task_prompt}")
    print(f"Specified task: {role_play.specified_task_prompt}")
    print(f"Final task: {role_play.task_prompt}")

    # 3. 执行对话
    input_msg = role_play.init_chat()
    conversation_history = []

    for turn in range(10):  # 最多10轮对话
        assistant_response, user_response = role_play.step(input_msg)

        # 记录对话
        conversation_history.append({
            "turn": turn + 1,
            "assistant": assistant_response.msgs[0].content,
            "user": user_response.msgs[0].content
        })

        print(f"\n--- Turn {turn + 1} ---")
        print(f"Assistant: {assistant_response.msgs[0].content}")
        print(f"User: {user_response.msgs[0].content}")

        # 检查终止条件
        if assistant_response.terminated or user_response.terminated:
            print("Conversation terminated.")
            break

        input_msg = assistant_response.msg

    return conversation_history

# 运行会话
history = run_role_playing_session()
```

### 3.2 带评论家的角色扮演

```python
from camel.societies import RolePlaying

def run_critic_role_playing():
    """运行带评论家的角色扮演会话"""

    role_play = RolePlaying(
        assistant_role_name="Content Writer",
        user_role_name="Editor",
        critic_role_name="Quality Reviewer",
        task_prompt="Write a blog post about artificial intelligence trends",
        with_critic_in_the_loop=True,  # 启用评论家循环
        critic_criteria="""Evaluate the content for:
        1. Accuracy and factual correctness
        2. Readability and engagement
        3. Structure and organization
        4. SEO optimization
        5. Target audience appropriateness""",
        model=model
    )

    input_msg = role_play.init_chat()

    for turn in range(15):
        assistant_response, user_response = role_play.step(input_msg)

        print(f"\n--- Turn {turn + 1} ---")
        print(f"Writer: {assistant_response.msgs[0].content}")
        print(f"Editor: {user_response.msgs[0].content}")

        # 检查是否有评论家的反馈
        if hasattr(assistant_response, 'critic_feedback'):
            print(f"Critic: {assistant_response.critic_feedback}")

        if assistant_response.terminated or user_response.terminated:
            break

        input_msg = assistant_response.msg

# 运行带评论家的会话
run_critic_role_playing()
```

### 3.3 Workforce工作力系统

```python
from camel.societies import Workforce
from camel.societies.workforce.workers import SingleAgentWorker
from camel.toolkits import SearchToolkit, CodeExecutionToolkit

def run_workforce_example():
    """运行工作力系统示例"""

    # 1. 创建专业化的工作者
    search_worker = SingleAgentWorker(
        agent=ChatAgent(
            system_message="You are a research specialist focused on web search.",
            tools=[FunctionTool(SearchToolkit().search_brave)]
        ),
        description="Search specialist for web research"
    )

    code_worker = SingleAgentWorker(
        agent=ChatAgent(
            system_message="You are a Python developer specializing in data analysis.",
            tools=[FunctionTool(CodeExecutionToolkit().execute_python)]
        ),
        description="Python developer for data analysis"
    )

    analysis_worker = SingleAgentWorker(
        agent=ChatAgent(
            system_message="You are a data analyst who creates insights from processed data."
        ),
        description="Data analyst for creating insights"
    )

    # 2. 创建工作力系统
    workforce = Workforce(
        workers=[search_worker, code_worker, analysis_worker],
        task_prompt="Analyze the current trends in artificial intelligence and create a summary report with data visualization",
        storage=None  # 可以添加存储系统
    )

    # 3. 执行任务
    print("Starting workforce task execution...")
    result = workforce.run()

    print(f"Task completed successfully: {result.success}")
    print(f"Final result: {result.final_result}")
    print(f"Execution time: {result.execution_time} seconds")

    return result

# 运行工作力示例
workforce_result = run_workforce_example()
```

### 3.4 BabyAGI自主系统

```python
from camel.societies import BabyAGI
from camel.agents import TaskCreationAgent, TaskPrioritizationAgent

def run_babyagi_example():
    """运行BabyAGI自主系统示例"""

    # 1. 定义目标
    objective = "Research and summarize the latest developments in quantum computing"

    # 2. 创建专业智能体
    creation_agent = TaskCreationAgent(
        system_message="You are a task creation agent that generates new tasks based on current objectives and results."
    )

    prioritization_agent = TaskPrioritizationAgent(
        system_message="You are a task prioritization agent that ranks tasks by importance."
    )

    execution_agent = ChatAgent(
        system_message="You are a research assistant that can search for information and create summaries.",
        tools=[FunctionTool(SearchToolkit().search_brave)]
    )

    # 3. 创建BabyAGI系统
    babyagi = BabyAGI(
        objective=objective,
        creation_agent=creation_agent,
        prioritization_agent=prioritization_agent,
        execution_agent=execution_agent,
        memory=ChatHistoryMemory(window_size=100)
    )

    # 4. 运行自主系统
    print(f"Starting BabyAGI with objective: {objective}")
    result = babyagi.run(max_iterations=10)

    print(f"Completed tasks: {len(result.completed_tasks)}")
    print(f"Final state: {result.final_state}")

    return result

# 运行BabyAGI示例
babyagi_result = run_babyagi_example()
```

---

## 4. 工具集成

### 4.1 基础工具使用

```python
from camel.toolkits import SearchToolkit, MathToolkit, CodeExecutionToolkit
from camel.toolkits.function_tool import FunctionTool

# 1. 创建工具实例
search_toolkit = SearchToolkit()
math_toolkit = MathToolkit()
code_toolkit = CodeExecutionToolkit()

# 2. 将工具包装为FunctionTool
search_tool = FunctionTool(search_toolkit.search_brave)
math_tool = FunctionTool(math_toolkit.calculate)
code_tool = FunctionTool(code_toolkit.execute_python)

# 3. 创建带工具的智能体
agent = ChatAgent(
    system_message="""You are a helpful assistant with access to search,
    math calculation, and code execution capabilities.""",
    tools=[search_tool, math_tool, code_tool]
)

# 4. 测试工具调用
response = agent.step("What is 15 * 23 + 47? Show me the calculation.")
print(response.msgs[0].content)

response = agent.step("Search for the latest news about quantum computing.")
print(response.msgs[0].content)

response = agent.step("Write and execute Python code to calculate the factorial of 10.")
print(response.msgs[0].content)
```

### 4.2 自定义工具开发

```python
from camel.toolkits import BaseToolkit
from camel.toolkits.function_tool import FunctionTool
import requests
import json

class WeatherToolkit(BaseToolkit):
    """天气查询工具包"""

    def __init__(self, api_key: str):
        super().__init__()
        self.api_key = api_key
        self.base_url = "http://api.openweathermap.org/data/2.5"

        # 注册工具
        self.add_tools([
            FunctionTool(self.get_current_weather),
            FunctionTool(self.get_weather_forecast),
            FunctionTool(self.get_air_quality)
        ])

    def get_current_weather(self, city: str, units: str = "metric") -> str:
        """获取当前天气"""
        try:
            url = f"{self.base_url}/weather"
            params = {
                "q": city,
                "appid": self.api_key,
                "units": units
            }

            response = requests.get(url, params=params)
            data = response.json()

            if response.status_code == 200:
                weather = {
                    "city": data["name"],
                    "temperature": data["main"]["temp"],
                    "description": data["weather"][0]["description"],
                    "humidity": data["main"]["humidity"],
                    "pressure": data["main"]["pressure"]
                }
                return json.dumps(weather, indent=2)
            else:
                return f"Error: {data.get('message', 'Unknown error')}"

        except Exception as e:
            return f"Weather API error: {str(e)}"

    def get_weather_forecast(self, city: str, days: int = 5) -> str:
        """获取天气预报"""
        try:
            url = f"{self.base_url}/forecast"
            params = {
                "q": city,
                "appid": self.api_key,
                "units": "metric",
                "cnt": days * 8  # 每天8个时间点（3小时间隔）
            }

            response = requests.get(url, params=params)
            data = response.json()

            if response.status_code == 200:
                forecast = []
                for item in data["list"][:days * 8]:
                    forecast.append({
                        "datetime": item["dt_txt"],
                        "temperature": item["main"]["temp"],
                        "description": item["weather"][0]["description"]
                    })
                return json.dumps(forecast, indent=2)
            else:
                return f"Error: {data.get('message', 'Unknown error')}"

        except Exception as e:
            return f"Forecast API error: {str(e)}"

    def get_air_quality(self, city: str) -> str:
        """获取空气质量"""
        try:
            # 这里使用简化的空气质量查询
            # 实际应用中需要使用专门的空气质量API
            return f"Air quality for {city}: Good (AQI: 45)"
        except Exception as e:
            return f"Air quality API error: {str(e)}"

# 使用自定义工具包
weather_toolkit = WeatherToolkit(api_key="your-openweather-api-key")
agent = ChatAgent(
    system_message="You are a weather assistant that can provide current weather, forecasts, and air quality information.",
    tools=weather_toolkit.get_tools()
)

response = agent.step("What's the current weather in Beijing?")
print(response.msgs[0].content)
```

### 4.3 高级工具模式

```python
from typing import Dict, Any, List
import asyncio
from camel.toolkits import BaseToolkit, FunctionTool

class AsyncDataProcessorToolkit(BaseToolkit):
    """异步数据处理工具包"""

    def __init__(self):
        super().__init__()
        self.add_tools([
            FunctionTool(self.process_csv_data),
            FunctionTool(self.analyze_data),
            FunctionTool(self.generate_report)
        ])

    async def process_csv_data(self, file_path: str, operations: List[str]) -> str:
        """异步处理CSV数据"""
        try:
            import pandas as pd

            # 异步读取CSV文件
            def read_csv():
                return pd.read_csv(file_path)

            # 在线程池中执行
            loop = asyncio.get_event_loop()
            df = await loop.run_in_executor(None, read_csv)

            # 执行操作
            results = {}
            for op in operations:
                if op == "summary":
                    results["summary"] = df.describe().to_dict()
                elif op == "columns":
                    results["columns"] = df.columns.tolist()
                elif op == "shape":
                    results["shape"] = df.shape
                elif op == "missing":
                    results["missing"] = df.isnull().sum().to_dict()

            return json.dumps(results, indent=2)

        except Exception as e:
            return f"Data processing error: {str(e)}"

    async def analyze_data(self, data_summary: Dict[str, Any], analysis_type: str) -> str:
        """异步分析数据"""
        try:
            if analysis_type == "basic":
                analysis = {
                    "data_type": "structured",
                    "recommendations": [
                        "Check for missing values",
                        "Analyze distributions",
                        "Look for correlations"
                    ]
                }
            elif analysis_type == "advanced":
                analysis = {
                    "data_type": "structured",
                    "statistical_tests": [
                        "Normality test",
                        "Correlation analysis",
                        "Outlier detection"
                    ],
                    "visualizations": [
                        "Histograms",
                        "Scatter plots",
                        "Box plots"
                    ]
                }
            else:
                analysis = {"error": "Unknown analysis type"}

            return json.dumps(analysis, indent=2)

        except Exception as e:
            return f"Data analysis error: {str(e)}"

    def generate_report(self, findings: List[str], format_type: str = "markdown") -> str:
        """生成分析报告"""
        try:
            if format_type == "markdown":
                report = "# Data Analysis Report\n\n"
                report += "## Key Findings\n\n"
                for i, finding in enumerate(findings, 1):
                    report += f"{i}. {finding}\n"

                report += "\n## Recommendations\n\n"
                report += "1. Data quality assessment\n"
                report += "2. Further analysis suggestions\n"
                report += "3. Action items\n"

            elif format_type == "json":
                report = json.dumps({
                    "findings": findings,
                    "recommendations": [
                        "Assess data quality",
                        "Perform deeper analysis",
                        "Create visualization"
                    ]
                }, indent=2)
            else:
                report = "Unsupported format type"

            return report

        except Exception as e:
            return f"Report generation error: {str(e)}"

# 使用异步工具包
async def use_async_tools():
    data_toolkit = AsyncDataProcessorToolkit()
    agent = ChatAgent(
        system_message="You are a data analysis assistant with advanced data processing capabilities.",
        tools=data_toolkit.get_tools()
    )

    response = await agent.astep("Process the sales data and generate a comprehensive analysis report.")
    print(response.msgs[0].content)

# 运行异步示例
asyncio.run(use_async_tools())
```

---

## 5. 记忆系统配置

### 5.1 基础记忆系统

```python
from camel.memories import ChatHistoryMemory, VectorDBMemory, LongtermAgentMemory

# 1. 聊天历史记忆
chat_memory = ChatHistoryMemory(
    window_size=100,           # 保存最近100条消息
    token_limit=4000,          # Token限制
    enable_tool_calling_messages=False  # 不保存工具调用消息
)

# 2. 向量数据库记忆（需要安装ChromaDB）
vector_memory = VectorDBMemory(
    embedding_model="text-embedding-ada-002",
    collection_name="agent_memory",
    similarity_threshold=0.7,   # 相似度阈值
    max_results=10             # 最大返回结果数
)

# 3. 长期记忆系统
longterm_memory = LongtermAgentMemory(
    short_term_memory=chat_memory,
    vector_memory=vector_memory,
    compression_threshold=50   # 压缩阈值
)

# 使用不同类型的记忆
agent_with_chat_memory = ChatAgent(
    system_message="You are a helpful assistant with short-term memory.",
    memory=chat_memory
)

agent_with_vector_memory = ChatAgent(
    system_message="You are a helpful assistant with semantic search memory.",
    memory=vector_memory
)

agent_with_longterm_memory = ChatAgent(
    system_message="You are a helpful assistant with comprehensive memory.",
    memory=longterm_memory
)
```

### 5.2 上下文创建策略

```python
from camel.memories import ScoreBasedContextCreator, AdaptiveContextCreator

# 1. 基于分数的上下文创建器
score_context_creator = ScoreBasedContextCreator(
    token_limit=4000,
    score_function=lambda record: record.importance_score,
    decay_factor=0.9,           # 指数衰减因子
    recency_weight=0.3,        # 新近性权重
    importance_weight=0.4,      # 重要性权重
    relevance_weight=0.3        # 相关性权重
)

# 2. 自适应上下文创建器
adaptive_context_creator = AdaptiveContextCreator(
    base_token_limit=4000,
    max_token_limit=8000,
    compression_threshold=0.8,
    complexity_threshold=0.7
)

# 3. 自定义上下文创建器
class CustomContextCreator:
    def __init__(self, max_recent_messages: int = 10):
        self.max_recent_messages = max_recent_messages

    def create_context(self, memories, current_message=None):
        # 获取最近的10条消息
        recent_memories = memories[-self.max_recent_messages:]

        # 如果有当前消息，进行语义搜索
        if current_message:
            # 这里可以添加语义搜索逻辑
            pass

        return [mem.msg for mem in recent_memories], 1000

# 使用自定义上下文创建器
agent = ChatAgent(
    system_message="You are a helpful assistant.",
    memory=ChatHistoryMemory(),
    context_creator=CustomContextCreator(max_recent_messages=15)
)
```

### 5.3 记忆系统高级配置

```python
from camel.memories import MemoryBlock, MemoryRecord
from typing import List, Optional
import datetime

class CustomMemoryBlock(MemoryBlock):
    """自定义记忆块"""

    def __init__(self, max_records: int = 1000):
        self.max_records = max_records
        self.records: List[MemoryRecord] = []
        self.index = {}  # 搜索索引

    def get_records(self, num: Optional[int] = None) -> List[MemoryRecord]:
        """获取记忆记录"""
        if num is None:
            return self.records.copy()
        elif num > 0:
            return self.records[-num:]
        else:
            return self.records[:num]

    def add_record(self, record: MemoryRecord) -> None:
        """添加记忆记录"""
        # 添加记录
        self.records.append(record)

        # 更新索引
        self._update_index(record)

        # 如果超过最大记录数，删除最旧的记录
        if len(self.records) > self.max_records:
            oldest_record = self.records.pop(0)
            self._remove_from_index(oldest_record)

    def _update_index(self, record: MemoryRecord):
        """更新搜索索引"""
        content = str(record.msg.content).lower()
        keywords = content.split()

        for keyword in keywords:
            if keyword not in self.index:
                self.index[keyword] = []
            self.index[keyword].append(record)

    def _remove_from_index(self, record: MemoryRecord):
        """从索引中移除记录"""
        content = str(record.msg.content).lower()
        keywords = content.split()

        for keyword in keywords:
            if keyword in self.index:
                if record in self.index[keyword]:
                    self.index[keyword].remove(record)

    def search_by_keyword(self, keyword: str) -> List[MemoryRecord]:
        """根据关键词搜索"""
        return self.index.get(keyword.lower(), [])

# 使用自定义记忆块
custom_memory = CustomMemoryBlock(max_records=500)
agent = ChatAgent(
    system_message="You are a helpful assistant with custom memory management.",
    memory=custom_memory
)
```

---

## 6. 模型管理优化

### 6.1 多模型负载均衡

```python
from camel.models import ModelManager, ModelFactory
from camel.types import ModelPlatformType, ModelType

# 1. 创建多个模型实例
openai_model = ModelFactory.create(
    model_platform=ModelPlatformType.OPENAI,
    model_type=ModelType.GPT_4O_MINI,
    model_config_dict={"temperature": 0.7}
)

anthropic_model = ModelFactory.create(
    model_platform=ModelPlatformType.ANTHROPIC,
    model_type=ModelType.CLAUDE_3_5_SONNET,
    model_config_dict={"temperature": 0.5}
)

google_model = ModelFactory.create(
    model_platform=ModelPlatformType.GOOGLE,
    model_type=ModelType.GEMINI_PRO,
    model_config_dict={"temperature": 0.6}
)

# 2. 创建模型管理器
model_manager = ModelManager(
    models=[openai_model, anthropic_model, google_model],
    scheduling_strategy="round_robin",  # 轮询调度
    health_check_interval=60.0          # 健康检查间隔
)

# 3. 使用模型管理器
agent = ChatAgent(
    system_message="You are a helpful assistant with multi-model support.",
    model=model_manager,
    retry_attempts=2,
    retry_delay=1.0
)

# 4. 监控模型性能
def monitor_model_performance():
    """监控模型性能"""
    for i in range(10):
        response = agent.step(f"Test message {i+1}")

        # 获取当前使用的模型
        current_model = model_manager.get_current_model()

        # 记录性能指标
        model_manager.update_model_stats(
            current_model,
            {
                "response_time": response.info.get("response_time", 0),
                "token_usage": response.info.get("usage", {}),
                "success": True
            }
        )

        print(f"Used model: {current_model.model_type}")
        print(f"Response time: {response.info.get('response_time', 0):.2f}s")

# 运行性能监控
monitor_model_performance()
```

### 6.2 智能模型选择

```python
from camel.models import ModelManager
from typing import Dict, Any

class SmartModelSelector:
    """智能模型选择器"""

    def __init__(self, model_manager: ModelManager):
        self.model_manager = model_manager
        self.task_model_mapping = {
            "code_generation": ["gpt-4o", "claude-3-5-sonnet"],
            "creative_writing": ["claude-3-opus", "gpt-4o"],
            "analysis": ["gpt-4o", "gemini-pro"],
            "translation": ["gpt-4o-mini", "claude-3-haiku"],
            "math": ["gpt-4o", "claude-3-5-sonnet"]
        }

    def select_model_for_task(self, task_type: str, complexity: str = "medium") -> str:
        """根据任务类型选择模型"""
        recommended_models = self.task_model_mapping.get(task_type, ["gpt-4o-mini"])

        if complexity == "high":
            return recommended_models[0]  # 使用最佳模型
        elif complexity == "low":
            return recommended_models[-1]  # 使用最经济的模型
        else:
            return recommended_models[len(recommended_models)//2]  # 使用中等模型

    def get_model_performance_stats(self) -> Dict[str, Any]:
        """获取模型性能统计"""
        stats = {}
        for model in self.model_manager.models:
            model_stats = self.model_manager.model_stats.get(model, {})
            stats[model.model_type] = {
                "total_calls": model_stats.get("total_calls", 0),
                "success_rate": model_stats.get("success_rate", 0.0),
                "average_latency": model_stats.get("average_latency", 0.0),
                "cost_per_call": model_stats.get("cost_per_call", 0.0)
            }
        return stats

# 使用智能模型选择器
smart_selector = SmartModelSelector(model_manager)

def process_task_with_smart_model(task_description: str):
    """使用智能模型选择处理任务"""
    # 分析任务类型
    task_type = "general"  # 这里可以添加任务类型分析逻辑
    complexity = "medium"

    # 选择模型
    selected_model = smart_selector.select_model_for_task(task_type, complexity)

    # 创建临时智能体
    temp_agent = ChatAgent(
        system_message="You are a helpful assistant.",
        model=ModelFactory.create("openai", selected_model)
    )

    # 处理任务
    response = temp_agent.step(task_description)

    print(f"Used model: {selected_model}")
    print(f"Response: {response.msgs[0].content}")

    return response

# 测试智能模型选择
process_task_with_smart_model("Write a Python function to calculate fibonacci numbers")
```

### 6.3 模型成本优化

```python
from camel.models import ModelManager, ModelFactory
from typing import List, Dict
import time

class CostOptimizedModelManager:
    """成本优化的模型管理器"""

    def __init__(self):
        # 定义模型成本（每1000 tokens）
        self.model_costs = {
            "gpt-4o": {"input": 0.005, "output": 0.015},
            "gpt-4o-mini": {"input": 0.00015, "output": 0.0006},
            "claude-3-5-sonnet": {"input": 0.003, "output": 0.015},
            "claude-3-haiku": {"input": 0.00025, "output": 0.00125},
            "gemini-pro": {"input": 0.00125, "output": 0.005}
        }

        # 创建模型实例
        self.models = {
            "gpt-4o": ModelFactory.create("openai", "gpt-4o"),
            "gpt-4o-mini": ModelFactory.create("openai", "gpt-4o-mini"),
            "claude-3-5-sonnet": ModelFactory.create("anthropic", "claude-3-5-sonnet"),
            "claude-3-haiku": ModelFactory.create("anthropic", "claude-3-haiku"),
            "gemini-pro": ModelFactory.create("google", "gemini-pro")
        }

        self.usage_tracking = {model: {"calls": 0, "tokens": 0, "cost": 0.0}
                             for model in self.models.keys()}

    def get_cheapest_model(self, task_complexity: str = "low") -> str:
        """获取最便宜的模型"""
        if task_complexity == "low":
            return "gpt-4o-mini"
        elif task_complexity == "medium":
            return "claude-3-haiku"
        else:
            return "gpt-4o"

    def estimate_cost(self, model: str, input_tokens: int, output_tokens: int) -> float:
        """估算成本"""
        costs = self.model_costs[model]
        input_cost = (input_tokens / 1000) * costs["input"]
        output_cost = (output_tokens / 1000) * costs["output"]
        return input_cost + output_cost

    def track_usage(self, model: str, tokens_used: int, response_time: float):
        """追踪使用情况"""
        self.usage_tracking[model]["calls"] += 1
        self.usage_tracking[model]["tokens"] += tokens_used

        # 估算成本
        estimated_cost = self.estimate_cost(model, tokens_used//2, tokens_used//2)
        self.usage_tracking[model]["cost"] += estimated_cost

    def get_cost_report(self) -> Dict[str, Any]:
        """获取成本报告"""
        total_cost = sum(stats["cost"] for stats in self.usage_tracking.values())
        total_calls = sum(stats["calls"] for stats in self.usage_tracking.values())

        return {
            "total_cost": total_cost,
            "total_calls": total_calls,
            "average_cost_per_call": total_cost / total_calls if total_calls > 0 else 0,
            "model_breakdown": self.usage_tracking,
            "most_used_model": max(self.usage_tracking.keys(),
                                 key=lambda x: self.usage_tracking[x]["calls"]),
            "most_cost_effective": min(self.usage_tracking.keys(),
                                     key=lambda x: self.usage_tracking[x]["cost"] / max(1, self.usage_tracking[x]["calls"]))
        }

# 使用成本优化的模型管理器
cost_manager = CostOptimizedModelManager()

def process_task_with_cost_optimization(task: str, complexity: str = "medium"):
    """使用成本优化处理任务"""
    # 选择模型
    selected_model = cost_manager.get_cheapest_model(complexity)
    model_instance = cost_manager.models[selected_model]

    # 创建智能体
    agent = ChatAgent(
        system_message="You are a helpful assistant.",
        model=model_instance
    )

    # 处理任务并计时
    start_time = time.time()
    response = agent.step(task)
    end_time = time.time()

    # 追踪使用情况
    estimated_tokens = len(task.split()) + len(response.msgs[0].content.split())
    cost_manager.track_usage(selected_model, estimated_tokens, end_time - start_time)

    return response

# 测试成本优化
tasks = [
    ("What is 2+2?", "low"),
    ("Explain quantum computing", "medium"),
    ("Write a Python function for quicksort", "high")
]

for task, complexity in tasks:
    response = process_task_with_cost_optimization(task, complexity)
    print(f"Task: {task}")
    print(f"Response: {response.msgs[0].content[:100]}...")
    print()

# 生成成本报告
cost_report = cost_manager.get_cost_report()
print("Cost Report:")
print(f"Total Cost: ${cost_report['total_cost']:.4f}")
print(f"Total Calls: {cost_report['total_calls']}")
print(f"Average Cost per Call: ${cost_report['average_cost_per_call']:.4f}")
```

---

## 7. 任务管理系统

### 7.1 基础任务管理

```python
from camel.tasks import Task, TaskManager
from camel.types import TaskStatus, TaskType

# 1. 创建任务
main_task = Task(
    content="Develop a comprehensive data analysis system",
    id="main_analysis_task",
    status=TaskStatus.TODO,
    metadata={
        "priority": "high",
        "estimated_hours": 40,
        "skills_required": ["Python", "Data Analysis", "Machine Learning"]
    }
)

# 2. 创建子任务
subtasks = [
    Task(
        content="Set up data collection pipeline",
        id="data_collection",
        parent_id="main_analysis_task",
        status=TaskStatus.TODO
    ),
    Task(
        content="Implement data cleaning functions",
        id="data_cleaning",
        parent_id="main_analysis_task",
        status=TaskStatus.TODO
    ),
    Task(
        content="Create analysis algorithms",
        id="analysis_algorithms",
        parent_id="main_analysis_task",
        status=TaskStatus.TODO
    ),
    Task(
        content="Build visualization dashboard",
        id="visualization",
        parent_id="main_analysis_task",
        status=TaskStatus.TODO
    )
]

# 3. 添加子任务到主任务
for subtask in subtasks:
    main_task.add_subtask(subtask)

# 4. 创建任务管理器
task_manager = TaskManager(main_task)

# 5. 分解任务
print("Task Decomposition:")
decomposed_tasks = task_manager.decompose(main_task, agent=analysis_agent)
for i, task in enumerate(decomposed_tasks, 1):
    print(f"{i}. {task.content} (Status: {task.status.value})")

# 6. 演化任务
print("\nTask Evolution:")
evolved_task = task_manager.evolve(main_task, agent=evolution_agent)
if evolved_task:
    print(f"Evolved task: {evolved_task.content}")
else:
    print("No evolution needed for this task.")

# 7. 任务状态管理
def update_task_status(task_manager: TaskManager, task_id: str, status: TaskStatus):
    """更新任务状态"""
    # 这里应该实现状态更新逻辑
    print(f"Task {task_id} status updated to: {status.value}")

# 执行任务
def execute_task_sequence(task_manager: TaskManager):
    """执行任务序列"""
    tasks = task_manager.get_tasks_by_status(TaskStatus.TODO)

    for task in tasks:
        print(f"Executing task: {task.content}")

        # 模拟任务执行
        update_task_status(task_manager, task.id, TaskStatus.RUNNING)

        # 这里应该有实际的执行逻辑
        time.sleep(1)  # 模拟执行时间

        update_task_status(task_manager, task.id, TaskStatus.DONE)
        print(f"Task {task.id} completed successfully.")

# 执行任务序列
execute_task_sequence(task_manager)
```

### 7.2 高级任务模式

```python
from camel.tasks import Task, TaskManager
from typing import List, Dict, Any
import uuid

class WorkflowTaskManager:
    """工作流任务管理器"""

    def __init__(self):
        self.tasks: Dict[str, Task] = {}
        self.dependencies: Dict[str, List[str]] = {}
        self.workflow_states: Dict[str, str] = {}

    def create_workflow(self, workflow_definition: Dict[str, Any]) -> str:
        """创建工作流"""
        workflow_id = str(uuid.uuid4())

        # 创建任务
        for task_def in workflow_definition["tasks"]:
            task = Task(
                content=task_def["content"],
                id=task_def["id"],
                metadata=task_def.get("metadata", {})
            )
            self.tasks[task.id] = task

            # 设置依赖关系
            if "depends_on" in task_def:
                self.dependencies[task.id] = task_def["depends_on"]

        self.workflow_states[workflow_id] = "created"
        return workflow_id

    def execute_workflow(self, workflow_id: str, agent: ChatAgent) -> Dict[str, Any]:
        """执行工作流"""
        if workflow_id not in self.workflow_states:
            raise ValueError(f"Workflow {workflow_id} not found")

        self.workflow_states[workflow_id] = "running"

        try:
            # 拓扑排序确定执行顺序
            execution_order = self._topological_sort()

            results = {}
            for task_id in execution_order:
                task = self.tasks[task_id]

                print(f"Executing task: {task.content}")

                # 执行任务
                response = agent.step(task.content)

                # 记录结果
                results[task_id] = {
                    "content": response.msgs[0].content,
                    "success": True,
                    "execution_time": response.info.get("response_time", 0)
                }

                print(f"Task {task_id} completed")

            self.workflow_states[workflow_id] = "completed"
            return {"workflow_id": workflow_id, "status": "success", "results": results}

        except Exception as e:
            self.workflow_states[workflow_id] = "failed"
            return {"workflow_id": workflow_id, "status": "failed", "error": str(e)}

    def _topological_sort(self) -> List[str]:
        """拓扑排序"""
        # 简化的拓扑排序实现
        in_degree = {task_id: 0 for task_id in self.tasks.keys()}

        # 计算入度
        for task_id, deps in self.dependencies.items():
            for dep in deps:
                if dep in in_degree:
                    in_degree[dep] += 1

        # 拓扑排序
        queue = [task_id for task_id, degree in in_degree.items() if degree == 0]
        result = []

        while queue:
            current = queue.pop(0)
            result.append(current)

            # 检查依赖当前任务的任务
            for task_id, deps in self.dependencies.items():
                if current in deps:
                    in_degree[task_id] -= 1
                    if in_degree[task_id] == 0:
                        queue.append(task_id)

        return result

# 定义工作流
data_analysis_workflow = {
    "tasks": [
        {
            "id": "collect_data",
            "content": "Collect sales data from various sources",
            "metadata": {"type": "data_collection", "estimated_time": "2 hours"}
        },
        {
            "id": "clean_data",
            "content": "Clean and preprocess the collected data",
            "depends_on": ["collect_data"],
            "metadata": {"type": "data_processing", "estimated_time": "1 hour"}
        },
        {
            "id": "analyze_data",
            "content": "Perform statistical analysis on the cleaned data",
            "depends_on": ["clean_data"],
            "metadata": {"type": "analysis", "estimated_time": "3 hours"}
        },
        {
            "id": "create_report",
            "content": "Create comprehensive analysis report",
            "depends_on": ["analyze_data"],
            "metadata": {"type": "reporting", "estimated_time": "2 hours"}
        },
        {
            "id": "create_visualization",
            "content": "Create data visualization dashboard",
            "depends_on": ["analyze_data"],
            "metadata": {"type": "visualization", "estimated_time": "1.5 hours"}
        }
    ]
}

# 使用工作流管理器
workflow_manager = WorkflowTaskManager()
workflow_id = workflow_manager.create_workflow(data_analysis_workflow)

# 执行工作流
agent = ChatAgent(
    system_message="You are a data analysis assistant that can help with various data processing tasks."
)

workflow_result = workflow_manager.execute_workflow(workflow_id, agent)

print(f"Workflow Result: {workflow_result['status']}")
if workflow_result['status'] == 'success':
    for task_id, result in workflow_result['results'].items():
        print(f"Task {task_id}: {result['content'][:100]}...")
```

### 7.3 动态任务生成

```python
from camel.tasks import Task, TaskManager
from camel.agents import TaskCreationAgent, TaskPrioritizationAgent
from typing import List, Dict, Any

class DynamicTaskGenerator:
    """动态任务生成器"""

    def __init__(self):
        self.creation_agent = TaskCreationAgent(
            system_message="You are a task creation agent that generates new tasks based on current objectives."
        )

        self.prioritization_agent = TaskPrioritizationAgent(
            system_message="You are a task prioritization agent that ranks tasks by importance."
        )

        self.task_queue = []
        self.completed_tasks = []
        self.objectives = []

    def add_objective(self, objective: str):
        """添加目标"""
        self.objectives.append(objective)
        self._generate_tasks_for_objective(objective)

    def _generate_tasks_for_objective(self, objective: str):
        """为目标生成任务"""
        prompt = f"""
        Given the objective: "{objective}"

        Generate 3-5 specific, actionable tasks to achieve this objective.
        Each task should be:
        1. Specific and measurable
        2. Achievable within a reasonable timeframe
        3. Relevant to the objective
        4. Time-bound where possible

        Format your response as a JSON array of task objects.
        """

        response = self.creation_agent.step(prompt)

        try:
            # 解析生成的任务
            import json
            tasks_data = json.loads(response.msgs[0].content)

            for task_data in tasks_data:
                task = Task(
                    content=task_data["content"],
                    id=str(uuid.uuid4()),
                    metadata={
                        "objective": objective,
                        "priority": task_data.get("priority", "medium"),
                        "estimated_time": task_data.get("estimated_time", "1 hour")
                    }
                )
                self.task_queue.append(task)

        except Exception as e:
            print(f"Error generating tasks: {e}")

    def prioritize_tasks(self) -> List[Task]:
        """优先级排序任务"""
        if not self.task_queue:
            return []

        # 创建任务描述
        task_descriptions = []
        for task in self.task_queue:
            task_descriptions.append(f"Task: {task.content}\nPriority: {task.metadata.get('priority', 'medium')}")

        prompt = f"""
        Given these tasks:
        {chr(10).join(task_descriptions)}

        Rank them by importance and urgency. Consider:
        1. Dependencies between tasks
        2. Time sensitivity
        3. Resource requirements
        4. Impact on objectives

        Return a JSON array with task IDs in order of priority.
        """

        response = self.prioritization_agent.step(prompt)

        try:
            # 重新排序任务队列
            import json
            priority_order = json.loads(response.msgs[0].content)

            # 根据优先级重新排序
            task_dict = {task.id: task for task in self.task_queue}
            self.task_queue = [task_dict[task_id] for task_id in priority_order if task_id in task_dict]

        except Exception as e:
            print(f"Error prioritizing tasks: {e}")

        return self.task_queue.copy()

    def get_next_task(self) -> Optional[Task]:
        """获取下一个任务"""
        if not self.task_queue:
            return None

        return self.task_queue.pop(0)

    def complete_task(self, task: Task, result: str):
        """完成任务"""
        self.completed_tasks.append({
            "task": task,
            "result": result,
            "completed_at": datetime.now()
        })

        # 检查是否需要生成新任务
        if "generate_followup" in task.metadata:
            self._generate_followup_tasks(task, result)

    def _generate_followup_tasks(self, completed_task: Task, result: str):
        """生成后续任务"""
        prompt = f"""
        Given the completed task: "{completed_task.content}"
        And the result: "{result}"

        Generate 1-2 follow-up tasks that would be logical next steps.
        Consider:
        1. What additional work is needed based on this result?
        2. What new insights or actions does this suggest?
        3. What dependencies or prerequisites have been established?
        """

        response = self.creation_agent.step(prompt)

        try:
            import json
            followup_tasks = json.loads(response.msgs[0].content)

            for task_data in followup_tasks:
                task = Task(
                    content=task_data["content"],
                    id=str(uuid.uuid4()),
                    metadata={
                        "followup_to": completed_task.id,
                        "priority": task_data.get("priority", "medium"),
                        "generated_from": "result_analysis"
                    }
                )
                self.task_queue.append(task)

        except Exception as e:
            print(f"Error generating follow-up tasks: {e}")

    def get_progress_summary(self) -> Dict[str, Any]:
        """获取进度摘要"""
        return {
            "total_objectives": len(self.objectives),
            "pending_tasks": len(self.task_queue),
            "completed_tasks": len(self.completed_tasks),
            "completion_rate": len(self.completed_tasks) / (len(self.completed_tasks) + len(self.task_queue)) if self.task_queue else 1.0,
            "recent_results": [task["result"][:100] for task in self.completed_tasks[-3:]]
        }

# 使用动态任务生成器
task_generator = DynamicTaskGenerator()

# 添加目标
task_generator.add_objective("Create a comprehensive machine learning model for customer churn prediction")
task_generator.add_objective("Develop a data pipeline for real-time analytics")

# 优先级排序任务
prioritized_tasks = task_generator.prioritize_tasks()
print(f"Prioritized tasks: {len(prioritized_tasks)}")

# 执行任务
agent = ChatAgent(
    system_message="You are a machine learning engineer assistant."
)

while True:
    next_task = task_generator.get_next_task()
    if not next_task:
        break

    print(f"Executing: {next_task.content}")
    response = agent.step(next_task.content)

    # 完成任务
    task_generator.complete_task(next_task, response.msgs[0].content)

    # 重新优先级排序
    task_generator.prioritize_tasks()

# 显示进度摘要
progress = task_generator.get_progress_summary()
print(f"Progress Summary:")
print(f"Completion Rate: {progress['completion_rate']:.2%}")
print(f"Completed Tasks: {progress['completed_tasks']}")
print(f"Pending Tasks: {progress['pending_tasks']}")
```

---

## 8. 高级应用场景

### 8.1 多语言对话系统

```python
from camel.societies import RolePlaying
from camel.agents import ChatAgent
from camel.models import ModelFactory
from camel.types import ModelPlatformType, ModelType

class MultiLanguageConversationSystem:
    """多语言对话系统"""

    def __init__(self):
        self.model = ModelFactory.create(
            model_platform=ModelPlatformType.OPENAI,
            model_type=ModelType.GPT_4O,
            model_config_dict={"temperature": 0.3}
        )

        self.language_agents = {}
        self.translator_agents = {}

    def create_language_agent(self, language: str, role_name: str, system_message: str):
        """创建语言专用智能体"""
        agent = ChatAgent(
            system_message=system_message,
            model=self.model
        )
        self.language_agents[language] = {
            "agent": agent,
            "role_name": role_name
        }

    def create_translator_agent(self, source_lang: str, target_lang: str):
        """创建翻译智能体"""
        translator = ChatAgent(
            system_message=f"""You are a professional translator. Translate text from {source_lang} to {target_lang}.
            Maintain the original meaning, tone, and context. Provide only the translation without additional commentary.""",
            model=self.model
        )

        key = f"{source_lang}_{target_lang}"
        self.translator_agents[key] = translator

    def translate_text(self, text: str, source_lang: str, target_lang: str) -> str:
        """翻译文本"""
        key = f"{source_lang}_{target_lang}"
        if key not in self.translator_agents:
            self.create_translator_agent(source_lang, target_lang)

        translator = self.translator_agents[key]
        response = translator.step(text)
        return response.msgs[0].content

    def multilingual_conversation(self, user_lang: str, assistant_lang: str, initial_message: str, turns: int = 5):
        """多语言对话"""
        if user_lang not in self.language_agents:
            self.create_language_agent(user_lang, "User", f"You are a user communicating in {user_lang}.")

        if assistant_lang not in self.language_agents:
            self.create_language_agent(assistant_lang, "Assistant", f"You are a helpful assistant responding in {assistant_lang}.")

        user_agent = self.language_agents[user_lang]["agent"]
        assistant_agent = self.language_agents[assistant_lang]["agent"]

        current_message = initial_message
        conversation = []

        for turn in range(turns):
            # 用户响应
            if turn > 0:  # 第一个消息是用户输入，不需要生成
                user_response = user_agent.step(current_message)
                user_message = user_response.msgs[0].content

                # 翻译到助手语言
                if user_lang != assistant_lang:
                    translated_message = self.translate_text(user_message, user_lang, assistant_lang)
                else:
                    translated_message = user_message
            else:
                translated_message = initial_message

            # 助手响应
            assistant_response = assistant_agent.step(translated_message)
            assistant_message = assistant_response.msgs[0].content

            # 翻译回用户语言
            if assistant_lang != user_lang:
                translated_response = self.translate_text(assistant_message, assistant_lang, user_lang)
            else:
                translated_response = assistant_message

            # 记录对话
            conversation.append({
                "turn": turn + 1,
                "user_input": current_message if turn > 0 else initial_message,
                "user_message": user_message if turn > 0 else initial_message,
                "assistant_response": assistant_message,
                "translated_response": translated_response
            })

            print(f"Turn {turn + 1}:")
            print(f"User ({user_lang}): {user_message if turn > 0 else initial_message}")
            print(f"Assistant ({assistant_lang}): {assistant_message}")
            print(f"Translated to {user_lang}: {translated_response}")
            print("-" * 50)

            current_message = translated_response

        return conversation

# 使用多语言对话系统
ml_system = MultiLanguageConversationSystem()

# 配置语言
ml_system.create_language_agent("Chinese", "Chinese User", "你是一个说中文的用户。")
ml_system.create_language_agent("English", "English Assistant", "You are a helpful assistant responding in English.")
ml_system.create_language_agent("Japanese", "Japanese User", "あなたは日本語を話すユーザーです。")

# 中英对话
print("=== Chinese-English Conversation ===")
conversation = ml_system.multilingual_conversation(
    user_lang="Chinese",
    assistant_lang="English",
    initial_message="请介绍一下人工智能的发展历史",
    turns=3
)

# 日英对话
print("\n=== Japanese-English Conversation ===")
conversation = ml_system.multilingual_conversation(
    user_lang="Japanese",
    assistant_lang="English",
    initial_message="人工知能の未来について教えてください",
    turns=3
)
```

### 8.2 代码审查系统

```python
from camel.societies import RolePlaying
from camel.agents import ChatAgent, CriticAgent
from camel.toolkits import CodeExecutionToolkit
from camel.toolkits.function_tool import FunctionTool
import os

class CodeReviewSystem:
    """代码审查系统"""

    def __init__(self):
        self.model = ModelFactory.create("openai", "gpt-4o")
        self.code_execution_toolkit = CodeExecutionToolkit()

        # 创建专业智能体
        self.developer_agent = ChatAgent(
            system_message="""You are an experienced Python developer. Write clean, efficient,
            and well-documented code. Follow PEP 8 guidelines and include type hints where appropriate.""",
            model=self.model,
            tools=[FunctionTool(self.code_execution_toolkit.execute_python)]
        )

        self.reviewer_agent = ChatAgent(
            system_message="""You are a senior code reviewer. Focus on:
            1. Code quality and best practices
            2. Performance optimization
            3. Security vulnerabilities
            4. Bug detection
            5. Documentation quality
            Provide constructive feedback with specific suggestions.""",
            model=self.model
        )

        self.security_agent = ChatAgent(
            system_message="""You are a security specialist. Review code for:
            1. Input validation
            2. SQL injection vulnerabilities
            3. XSS vulnerabilities
            4. Authentication/authorization issues
            5. Data encryption
            Provide detailed security recommendations.""",
            model=self.model
        )

        self.critic_agent = CriticAgent(
            system_message="""You are a final quality assurance reviewer. Evaluate the overall
            code quality based on all feedback and provide a final assessment.""",
            model=self.model
        )

    def generate_code(self, requirements: str) -> str:
        """生成代码"""
        prompt = f"""
        Generate Python code based on these requirements:
        {requirements}

        Requirements:
        1. Write clean, modular code
        2. Include proper error handling
        3. Add type hints
        4. Include docstrings
        5. Write unit tests if applicable
        """

        response = self.developer_agent.step(prompt)
        return self._extract_code(response.msgs[0].content)

    def review_code(self, code: str) -> Dict[str, Any]:
        """审查代码"""
        reviews = {}

        # 代码质量审查
        quality_review = self.reviewer_agent.step(f"""
        Review this Python code for quality and best practices:

        ```python
        {code}
        ```

        Focus on readability, maintainability, and performance.
        """)
        reviews["quality"] = quality_review.msgs[0].content

        # 安全审查
        security_review = self.security_agent.step(f"""
        Review this Python code for security vulnerabilities:

        ```python
        {code}
        ```

        Identify potential security issues and provide recommendations.
        """)
        reviews["security"] = security_review.msgs[0].content

        # 功能测试
        test_result = self._test_code(code)
        reviews["functionality"] = test_result

        # 综合评估
       综合评估 = self.critic_agent.step(f"""
        Based on these reviews:

        Quality Review:
        {reviews["quality"]}

        Security Review:
        {reviews["security"]}

        Test Results:
        {reviews["functionality"]}

        Provide a final assessment of the code quality and recommendations.
        """)
        reviews["overall"] = 综合评估.msgs[0].content

        return reviews

    def _extract_code(self, response: str) -> str:
        """从响应中提取代码"""
        import re

        # 查找代码块
        code_blocks = re.findall(r'```python\n(.*?)\n```', response, re.DOTALL)

        if code_blocks:
            return code_blocks[0]
        else:
            # 如果没有找到代码块，返回整个响应
            return response

    def _test_code(self, code: str) -> str:
        """测试代码"""
        try:
            # 执行代码
            result = self.code_execution_toolkit.execute_python(code)
            return f"Code executed successfully:\n{result}"
        except Exception as e:
            return f"Code execution failed:\n{str(e)}"

    def iterative_improvement(self, requirements: str, max_iterations: int = 3) -> Dict[str, Any]:
        """迭代改进代码"""
        print(f"Starting code generation for: {requirements}")

        # 初始代码生成
        current_code = self.generate_code(requirements)
        print(f"Generated initial code ({len(current_code)} characters)")

        for iteration in range(max_iterations):
            print(f"\n--- Iteration {iteration + 1} ---")

            # 审查代码
            reviews = self.review_code(current_code)

            print("Reviews completed:")
            for review_type, review_content in reviews.items():
                print(f"{review_type}: {review_content[:100]}...")

            # 检查是否需要改进
            improvement_prompt = f"""
            Current code:
            ```python
            {current_code}
            ```

            Reviews:
            Quality: {reviews['quality']}
            Security: {reviews['security']}
            Functionality: {reviews['functionality']}

            Please improve the code based on these reviews. Address all issues mentioned.
            """

            # 生成改进的代码
            improvement_response = self.developer_agent.step(improvement_prompt)
            improved_code = self._extract_code(improvement_response.msgs[0].content)

            print(f"Code improved ({len(improved_code)} characters)")

            # 更新代码
            current_code = improved_code

        # 最终审查
        final_reviews = self.review_code(current_code)

        return {
            "final_code": current_code,
            "reviews": final_reviews,
            "iterations": max_iterations
        }

# 使用代码审查系统
code_review_system = CodeReviewSystem()

# 示例：生成和改进用户认证系统
requirements = """
Create a user authentication system with the following features:
1. User registration with email and password
2. User login with JWT tokens
3. Password hashing with bcrypt
4. Input validation
5. Basic rate limiting
"""

result = code_review_system.iterative_improvement(requirements, max_iterations=2)

print("\n=== Final Code ===")
print(result["final_code"])

print("\n=== Final Reviews ===")
for review_type, review_content in result["reviews"].items():
    print(f"{review_type.upper()}:")
    print(review_content)
    print("-" * 50)
```

### 8.3 智能客服系统

```python
from camel.agents import ChatAgent
from camel.societies import RolePlaying
from camel.memories import ChatHistoryMemory, VectorDBMemory
from camel.toolkits import SearchToolkit
from camel.toolkits.function_tool import FunctionTool
from typing import Dict, Any, List
import json
import time

class IntelligentCustomerService:
    """智能客服系统"""

    def __init__(self, company_name: str, knowledge_base: List[Dict[str, str]]):
        self.company_name = company_name
        self.model = ModelFactory.create("openai", "gpt-4o")

        # 初始化知识库
        self.knowledge_base = knowledge_base
        self.vector_memory = self._initialize_knowledge_base()

        # 创建客服智能体
        self.service_agent = ChatAgent(
            system_message=f"""You are a customer service representative for {company_name}.
            Your role is to:
            1. Provide accurate information based on the knowledge base
            2. Be empathetic and patient with customers
            3. Resolve issues efficiently
            4. Escalate complex issues when necessary
            5. Follow company policies and procedures

            Always maintain a professional and helpful tone.""",
            model=self.model,
            memory=ChatHistoryMemory(window_size=20),
            tools=[
                FunctionTool(SearchToolkit().search_brave),
                FunctionTool(self.search_knowledge_base),
                FunctionTool(self.create_ticket),
                FunctionTool(self.check_order_status),
                FunctionTool(self.process_refund)
            ]
        )

        # 创建主管智能体
        self.supervisor_agent = ChatAgent(
            system_message="""You are a customer service supervisor. Your role is to:
            1. Handle escalated customer issues
            2. Make decisions on complex cases
            3. Approve refunds and exceptions
            4. Provide guidance to service agents

            Always prioritize customer satisfaction while following company policies.""",
            model=self.model
        )

        # 客户会话状态
        self.customer_sessions = {}
        self.active_tickets = {}

    def _initialize_knowledge_base(self) -> VectorDBMemory:
        """初始化知识库向量存储"""
        # 这里简化了向量存储的实现
        # 实际应用中应该使用ChromaDB或类似系统
        return VectorDBMemory(
            embedding_model="text-embedding-ada-002",
            collection_name="customer_service_knowledge"
        )

    def search_knowledge_base(self, query: str) -> str:
        """搜索知识库"""
        # 模拟知识库搜索
        relevant_articles = []

        for article in self.knowledge_base:
            if any(keyword.lower() in query.lower() for keyword in article["keywords"]):
                relevant_articles.append(article)

        if relevant_articles:
            result = "Based on our knowledge base:\n\n"
            for article in relevant_articles[:3]:  # 返回最相关的3条
                result += f"Topic: {article['title']}\n"
                result += f"Content: {article['content'][:200]}...\n\n"

            return result
        else:
            return "No specific information found in knowledge base. Would you like me to search online for more information?"

    def create_ticket(self, customer_id: str, issue_type: str, description: str, priority: str = "medium") -> str:
        """创建服务票据"""
        ticket_id = f"TKT-{int(time.time())}"

        ticket = {
            "ticket_id": ticket_id,
            "customer_id": customer_id,
            "issue_type": issue_type,
            "description": description,
            "priority": priority,
            "status": "open",
            "created_at": time.strftime("%Y-%m-%d %H:%M:%S"),
            "assigned_to": "service_agent"
        }

        self.active_tickets[ticket_id] = ticket

        return f"Ticket {ticket_id} has been created successfully. Our team will respond within 24 hours."

    def check_order_status(self, order_id: str) -> str:
        """检查订单状态"""
        # 模拟订单状态检查
        mock_orders = {
            "ORD-12345": {"status": "shipped", "tracking_number": "TN123456789"},
            "ORD-67890": {"status": "processing", "estimated_delivery": "2024-01-15"},
            "ORD-24680": {"status": "delivered", "delivered_at": "2024-01-10"}
        }

        if order_id in mock_orders:
            order_info = mock_orders[order_id]
            response = f"Order {order_id} status: {order_info['status']}"

            if "tracking_number" in order_info:
                response += f"\nTracking number: {order_info['tracking_number']}"
            if "estimated_delivery" in order_info:
                response += f"\nEstimated delivery: {order_info['estimated_delivery']}"
            if "delivered_at" in order_info:
                response += f"\nDelivered at: {order_info['delivered_at']}"

            return response
        else:
            return f"Order {order_id} not found. Please check the order number and try again."

    def process_refund(self, order_id: str, reason: str, amount: float) -> str:
        """处理退款请求"""
        # 模拟退款处理
        if amount <= 0:
            return "Invalid refund amount. Please provide a positive amount."

        if amount > 1000:
            return f"Refund request for ${amount} requires supervisor approval. Creating ticket for review..."

        # 简化的退款逻辑
        refund_id = f"REF-{int(time.time())}"
        return f"Refund {refund_id} for order {order_id} has been processed successfully. Amount: ${amount:.2f}. Reason: {reason}"

    def start_customer_session(self, customer_id: str, customer_name: str) -> str:
        """开始客户会话"""
        session_id = f"SESSION-{customer_id}-{int(time.time())}"

        self.customer_sessions[session_id] = {
            "customer_id": customer_id,
            "customer_name": customer_name,
            "start_time": time.strftime("%Y-%m-%d %H:%M:%S"),
            "conversation_history": [],
            "issue_resolved": False,
            "satisfaction_rating": None
        }

        welcome_message = f"""
        Welcome to {self.company_name} Customer Service, {customer_name}!

        I'm your virtual assistant. How can I help you today?

        I can assist you with:
        • Product information and pricing
        • Order status and tracking
        • Technical support
        • Billing and refunds
        • Account management

        Please let me know how I can help you.
        """

        return welcome_message

    def handle_customer_message(self, session_id: str, message: str) -> str:
        """处理客户消息"""
        if session_id not in self.customer_sessions:
            return "Session not found. Please start a new session."

        session = self.customer_sessions[session_id]

        # 检查是否需要主管介入
        if self._needs_supervisor_escalation(message):
            return self._escalate_to_supervisor(session_id, message)

        # 正常客服响应
        response = self.service_agent.step(message)
        response_text = response.msgs[0].content

        # 记录对话
        session["conversation_history"].append({
            "timestamp": time.strftime("%Y-%m-%d %H:%M:%S"),
            "customer_message": message,
            "agent_response": response_text
        })

        return response_text

    def _needs_supervisor_escalation(self, message: str) -> bool:
        """检查是否需要主管介入"""
        escalation_keywords = [
            "supervisor", "manager", "escalate", "complaint", "lawyer",
            "refund over 1000", "urgent", "emergency"
        ]

        return any(keyword.lower() in message.lower() for keyword in escalation_keywords)

    def _escalate_to_supervisor(self, session_id: str, message: str) -> str:
        """升级到主管"""
        session = self.customer_sessions[session_id]

        supervisor_prompt = f"""
        Customer {session['customer_name']} (ID: {session['customer_id']}) has escalated an issue.

        Customer message: {message}

        Conversation history:
        {json.dumps(session['conversation_history'][-3:], indent=2)}

        Please handle this escalation appropriately.
        """

        supervisor_response = self.supervisor_agent.step(supervisor_prompt)

        # 记录升级
        session["conversation_history"].append({
            "timestamp": time.strftime("%Y-%m-%d %H:%M:%S"),
            "customer_message": message,
            "agent_response": supervisor_response.msgs[0].content,
            "escalated": True
        })

        return supervisor_response.msgs[0].content

    def end_session(self, session_id: str, satisfaction_rating: int = None) -> Dict[str, Any]:
        """结束会话"""
        if session_id not in self.customer_sessions:
            return {"error": "Session not found"}

        session = self.customer_sessions[session_id]

        # 更新会话信息
        session["end_time"] = time.strftime("%Y-%m-%d %H:%M:%S")
        session["satisfaction_rating"] = satisfaction_rating
        session["issue_resolved"] = self._check_issue_resolution(session)

        # 生成会话摘要
        summary_prompt = f"""
        Summarize this customer service session:

        Customer: {session['customer_name']}
        Duration: {session['start_time']} to {session['end_time']}
        Conversation turns: {len(session['conversation_history'])}

        Key conversation points:
        {json.dumps([conv['customer_message'][:100] for conv in session['conversation_history']], indent=2)}

        Provide a brief summary of the interaction and outcome.
        """

        summary_response = self.service_agent.step(summary_prompt)

        session_summary = {
            "session_id": session_id,
            "customer_id": session["customer_id"],
            "duration": self._calculate_duration(session["start_time"], session["end_time"]),
            "conversation_turns": len(session["conversation_history"]),
            "issue_resolved": session["issue_resolved"],
            "satisfaction_rating": satisfaction_rating,
            "summary": summary_response.msgs[0].content
        }

        # 清理会话
        del self.customer_sessions[session_id]

        return session_summary

    def _check_issue_resolution(self, session: Dict[str, Any]) -> bool:
        """检查问题是否已解决"""
        # 简化的解决检测逻辑
        recent_conversation = session["conversation_history"][-3:]  # 最近3轮对话

        resolution_keywords = ["resolved", "fixed", "solved", "thank you", "thanks", "appreciate"]

        for conv in recent_conversation:
            if any(keyword in conv["agent_response"].lower() for keyword in resolution_keywords):
                return True

        return False

    def _calculate_duration(self, start_time: str, end_time: str) -> str:
        """计算会话时长"""
        # 简化的时长计算
        return "Approximately 5-10 minutes"  # 实际应用中应该精确计算

    def get_service_metrics(self) -> Dict[str, Any]:
        """获取服务指标"""
        # 模拟服务指标
        return {
            "total_sessions": len(self.customer_sessions) + 50,  # 历史总数
            "active_tickets": len(self.active_tickets),
            "average_resolution_time": "8.5 minutes",
            "customer_satisfaction": "4.2/5.0",
            "first_contact_resolution": "78%",
            "escalation_rate": "12%"
        }

# 使用智能客服系统
knowledge_base = [
    {
        "title": "Product Returns",
        "content": "Customers can return products within 30 days of purchase. Items must be in original condition with proof of purchase.",
        "keywords": ["return", "refund", "30 days", "purchase"]
    },
    {
        "title": "Shipping Information",
        "content": "Standard shipping takes 3-5 business days. Express shipping is available for an additional fee.",
        "keywords": ["shipping", "delivery", "express", "business days"]
    },
    {
        "title": "Technical Support",
        "content": "Technical support is available 24/7 via phone and email. Response time is typically within 2 hours.",
        "keywords": ["technical", "support", "phone", "email", "24/7"]
    }
]

customer_service = IntelligentCustomerService(
    company_name="TechCorp Solutions",
    knowledge_base=knowledge_base
)

# 模拟客户服务会话
print("=== Customer Service Session ===")
session_id = "CUST-12345"
welcome = customer_service.start_customer_session(session_id, "John Doe")
print(welcome)

# 模拟对话
customer_messages = [
    "Hi, I'd like to check the status of my order ORD-12345",
    "I received it but there's a problem with the product",
    "The item is defective and I'd like to return it",
    "Thank you for your help with the return process"
]

for message in customer_messages:
    print(f"\nCustomer: {message}")
    response = customer_service.handle_customer_message(session_id, message)
    print(f"Agent: {response}")
    time.sleep(1)  # 模拟真实对话间隔

# 结束会话
session_summary = customer_service.end_session(session_id, satisfaction_rating=5)
print(f"\n=== Session Summary ===")
print(f"Duration: {session_summary['duration']}")
print(f"Issue Resolved: {session_summary['issue_resolved']}")
print(f"Satisfaction Rating: {session_summary['satisfaction_rating']}/5")
print(f"Summary: {session_summary['summary']}")

# 显示服务指标
metrics = customer_service.get_service_metrics()
print(f"\n=== Service Metrics ===")
for metric, value in metrics.items():
    print(f"{metric.replace('_', ' ').title()}: {value}")
```

---

## 9. 性能优化

### 9.1 异步处理优化

```python
import asyncio
from camel.agents import ChatAgent
from typing import List, Any
import time

class AsyncAgentManager:
    """异步智能体管理器"""

    def __init__(self, max_concurrent_tasks: int = 5):
        self.max_concurrent_tasks = max_concurrent_tasks
        self.semaphore = asyncio.Semaphore(max_concurrent_tasks)
        self.agents = {}

    def create_agent(self, agent_id: str, system_message: str, model_config: dict = None) -> ChatAgent:
        """创建智能体"""
        if model_config is None:
            model_config = {"model_platform": "openai", "model_type": "gpt-4o-mini"}

        agent = ChatAgent(
            system_message=system_message,
            model=ModelFactory.create(**model_config)
        )

        self.agents[agent_id] = agent
        return agent

    async def process_task_async(self, agent_id: str, task: str) -> Any:
        """异步处理任务"""
        async with self.semaphore:
            if agent_id not in self.agents:
                raise ValueError(f"Agent {agent_id} not found")

            agent = self.agents[agent_id]
            response = await agent.astep(task)
            return response

    async def process_multiple_tasks(self, tasks: List[dict]) -> List[Any]:
        """并行处理多个任务"""
        task_coroutines = []

        for task_data in tasks:
            coro = self.process_task_async(
                task_data["agent_id"],
                task_data["task"]
            )
            task_coroutines.append(coro)

        # 并行执行所有任务
        results = await asyncio.gather(*task_coroutines, return_exceptions=True)

        # 处理结果
        processed_results = []
        for i, result in enumerate(results):
            if isinstance(result, Exception):
                processed_results.append({
                    "task": tasks[i]["task"],
                    "error": str(result),
                    "status": "failed"
                })
            else:
                processed_results.append({
                    "task": tasks[i]["task"],
                    "result": result.msgs[0].content,
                    "status": "success",
                    "response_time": result.info.get("response_time", 0)
                })

        return processed_results

# 使用异步智能体管理器
async def demonstrate_async_processing():
    """演示异步处理"""
    manager = AsyncAgentManager(max_concurrent_tasks=3)

    # 创建智能体
    manager.create_agent(
        "analyst",
        "You are a data analyst.",
        {"model_platform": "openai", "model_type": "gpt-4o-mini"}
    )

    manager.create_agent(
        "writer",
        "You are a content writer.",
        {"model_platform": "anthropic", "model_type": "claude-3-5-sonnet"}
    )

    manager.create_agent(
        "researcher",
        "You are a research assistant.",
        {"model_platform": "google", "model_type": "gemini-pro"}
    )

    # 创建任务列表
    tasks = [
        {"agent_id": "analyst", "task": "Analyze the sales trends for Q4 2023"},
        {"agent_id": "writer", "task": "Write a blog post about AI trends"},
        {"agent_id": "researcher", "task": "Research the latest developments in quantum computing"},
        {"agent_id": "analyst", "task": "Compare customer satisfaction scores"},
        {"agent_id": "writer", "task": "Create product descriptions for new items"}
    ]

    print("Starting parallel task processing...")
    start_time = time.time()

    # 并行处理任务
    results = await manager.process_multiple_tasks(tasks)

    end_time = time.time()
    total_time = end_time - start_time

    print(f"Completed {len(tasks)} tasks in {total_time:.2f} seconds")
    print(f"Average time per task: {total_time/len(tasks):.2f} seconds")

    # 显示结果
    for i, result in enumerate(results):
        print(f"\nTask {i+1} ({result['status']}):")
        print(f"Task: {result['task']}")
        if result['status'] == 'success':
            print(f"Response time: {result['response_time']:.2f}s")
            print(f"Result preview: {result['result'][:100]}...")
        else:
            print(f"Error: {result['error']}")

# 运行异步演示
asyncio.run(demonstrate_async_processing())
```

### 9.2 缓存机制

```python
from functools import wraps
from typing import Dict, Any, Optional
import hashlib
import json
import time

class AgentResponseCache:
    """智能体响应缓存"""

    def __init__(self, max_size: int = 1000, ttl: int = 3600):
        self.max_size = max_size
        self.ttl = ttl  # Time-to-live in seconds
        self.cache: Dict[str, Dict[str, Any]] = {}
        self.access_times: Dict[str, float] = {}

    def _generate_key(self, agent_id: str, message: str, **kwargs) -> str:
        """生成缓存键"""
        # 创建包含所有参数的键
        cache_data = {
            "agent_id": agent_id,
            "message": message,
            **kwargs
        }

        # 使用JSON序列化和MD5哈希
        json_str = json.dumps(cache_data, sort_keys=True)
        return hashlib.md5(json_str.encode()).hexdigest()

    def get(self, agent_id: str, message: str, **kwargs) -> Optional[Dict[str, Any]]:
        """获取缓存的响应"""
        key = self._generate_key(agent_id, message, **kwargs)

        if key not in self.cache:
            return None

        cached_data = self.cache[key]

        # 检查是否过期
        if time.time() - cached_data["timestamp"] > self.ttl:
            del self.cache[key]
            del self.access_times[key]
            return None

        # 更新访问时间
        self.access_times[key] = time.time()

        return cached_data["response"]

    def set(self, agent_id: str, message: str, response: Dict[str, Any], **kwargs):
        """设置缓存"""
        key = self._generate_key(agent_id, message, **kwargs)

        # 如果缓存已满，删除最久未使用的项
        if len(self.cache) >= self.max_size:
            oldest_key = min(self.access_times.keys(), key=lambda k: self.access_times[k])
            del self.cache[oldest_key]
            del self.access_times[oldest_key]

        # 存储缓存数据
        self.cache[key] = {
            "response": response,
            "timestamp": time.time()
        }

        self.access_times[key] = time.time()

    def clear(self):
        """清空缓存"""
        self.cache.clear()
        self.access_times.clear()

    def get_stats(self) -> Dict[str, Any]:
        """获取缓存统计信息"""
        return {
            "cache_size": len(self.cache),
            "max_size": self.max_size,
            "ttl": self.ttl,
            "hit_rate": self._calculate_hit_rate()
        }

    def _calculate_hit_rate(self) -> float:
        """计算缓存命中率"""
        # 简化的命中率计算
        # 实际应用中应该追踪命中和未命中次数
        return 0.75  # 模拟75%的命中率

def cached_agent_call(cache: AgentResponseCache):
    """缓存智能体调用的装饰器"""
    def decorator(func):
        @wraps(func)
        async def wrapper(agent_id: str, message: str, **kwargs):
            # 尝试从缓存获取
            cached_response = cache.get(agent_id, message, **kwargs)
            if cached_response:
                print(f"Cache hit for agent {agent_id}")
                return cached_response

            # 调用原始函数
            print(f"Cache miss for agent {agent_id}")
            result = await func(agent_id, message, **kwargs)

            # 缓存结果
            cache.set(agent_id, message, result, **kwargs)

            return result

        return wrapper
    return decorator

# 使用缓存机制
class CachedAgentManager:
    """缓存的智能体管理器"""

    def __init__(self):
        self.cache = AgentResponseCache(max_size=500, ttl=1800)  # 30分钟TTL
        self.agents = {}

    def add_agent(self, agent_id: str, agent: ChatAgent):
        """添加智能体"""
        self.agents[agent_id] = agent

    @cached_agent_call(cache=AgentResponseCache(max_size=500, ttl=1800))
    async def cached_step(self, agent_id: str, message: str) -> Any:
        """带缓存的智能体调用"""
        if agent_id not in self.agents:
            raise ValueError(f"Agent {agent_id} not found")

        agent = self.agents[agent_id]
        response = await agent.astep(message)
        return response

# 演示缓存效果
async def demonstrate_caching():
    """演示缓存效果"""
    manager = CachedAgentManager()

    # 添加智能体
    agent = ChatAgent("You are a helpful assistant.")
    manager.add_agent("assistant", agent)

    # 重复调用相同的问题
    questions = [
        "What is the capital of France?",
        "What is the capital of France?",  # 重复
        "Explain quantum computing in simple terms",
        "What is the capital of France?",  # 再次重复
        "Explain quantum computing in simple terms",  # 重复
    ]

    print("Demonstrating agent response caching...")
    start_time = time.time()

    for i, question in enumerate(questions):
        print(f"\nQuestion {i+1}: {question}")

        response = await manager.cached_step("assistant", question)
        print(f"Response: {response.msgs[0].content[:100]}...")

        # 显示缓存统计
        stats = manager.cache.get_stats()
        print(f"Cache stats: {stats['cache_size']}/{stats['max_size']} items, Hit rate: {stats['hit_rate']:.2%}")

    total_time = time.time() - start_time
    print(f"\nTotal time: {total_time:.2f} seconds")

    # 显示最终缓存统计
    final_stats = manager.cache.get_stats()
    print(f"Final cache stats: {final_stats}")

# 运行缓存演示
asyncio.run(demonstrate_caching())
```

### 9.3 连接池和资源管理

```python
import threading
import queue
import time
from contextlib import contextmanager
from typing import Dict, Any, Optional

class AgentConnectionPool:
    """智能体连接池"""

    def __init__(self, max_connections: int = 10, idle_timeout: int = 300):
        self.max_connections = max_connections
        self.idle_timeout = idle_timeout
        self.connections: Dict[str, queue.Queue] = {}
        self.connection_counts: Dict[str, int] = {}
        self.last_used: Dict[str, float] = {}
        self.lock = threading.Lock()

        # 启动清理线程
        self.cleanup_thread = threading.Thread(target=self._cleanup_idle_connections, daemon=True)
        self.cleanup_thread.start()

    def get_connection(self, agent_id: str, create_func) -> Any:
        """获取连接"""
        with self.lock:
            # 初始化连接队列
            if agent_id not in self.connections:
                self.connections[agent_id] = queue.Queue(maxsize=self.max_connections)
                self.connection_counts[agent_id] = 0
                self.last_used[agent_id] = time.time()

            # 尝试从队列获取连接
            try:
                connection = self.connections[agent_id].get_nowait()
                self.last_used[agent_id] = time.time()
                return connection
            except queue.Empty:
                # 如果队列不为空且连接数未达到上限，创建新连接
                if self.connection_counts[agent_id] < self.max_connections:
                    connection = create_func()
                    self.connection_counts[agent_id] += 1
                    self.last_used[agent_id] = time.time()
                    return connection
                else:
                    # 等待连接可用
                    try:
                        connection = self.connections[agent_id].get(timeout=30)
                        self.last_used[agent_id] = time.time()
                        return connection
                    except queue.Empty:
                        raise TimeoutError("No available connections")

    def return_connection(self, agent_id: str, connection: Any):
        """归还连接"""
        with self.lock:
            if agent_id in self.connections:
                try:
                    self.connections[agent_id].put_nowait(connection)
                    self.last_used[agent_id] = time.time()
                except queue.Full:
                    # 队列已满，丢弃连接
                    self.connection_counts[agent_id] -= 1

    def _cleanup_idle_connections(self):
        """清理空闲连接"""
        while True:
            time.sleep(60)  # 每分钟检查一次

            with self.lock:
                current_time = time.time()
                agents_to_remove = []

                for agent_id, last_used_time in self.last_used.items():
                    if current_time - last_used_time > self.idle_timeout:
                        agents_to_remove.append(agent_id)

                # 清理长时间未使用的连接
                for agent_id in agents_to_remove:
                    try:
                        while not self.connections[agent_id].empty():
                            self.connections[agent_id].get_nowait()

                        del self.connections[agent_id]
                        del self.connection_counts[agent_id]
                        del self.last_used[agent_id]

                        print(f"Cleaned up idle connections for agent {agent_id}")
                    except queue.Empty:
                        pass

class ResourceManager:
    """资源管理器"""

    def __init__(self):
        self.connection_pool = AgentConnectionPool(max_connections=5)
        self.memory_pools: Dict[str, list] = {}
        self.thread_pools: Dict[str, Any] = {}

    @contextmanager
    def get_agent_connection(self, agent_id: str, create_func):
        """获取智能体连接的上下文管理器"""
        connection = None
        try:
            connection = self.connection_pool.get_connection(agent_id, create_func)
            yield connection
        finally:
            if connection:
                self.connection_pool.return_connection(agent_id, connection)

    def create_memory_pool(self, pool_name: str, pool_size: int = 100):
        """创建内存池"""
        if pool_name not in self.memory_pools:
            self.memory_pools[pool_name] = []
            # 预分配内存
            for _ in range(pool_size):
                self.memory_pools[pool_name].append(bytearray(1024))  # 1KB内存块

    def get_memory_chunk(self, pool_name: str) -> Optional[bytearray]:
        """从内存池获取内存块"""
        if pool_name in self.memory_pools and self.memory_pools[pool_name]:
            return self.memory_pools[pool_name].pop()
        return None

    def return_memory_chunk(self, pool_name: str, chunk: bytearray):
        """归还内存块到池"""
        if pool_name in self.memory_pools:
            self.memory_pools[pool_name].append(chunk)

    def get_thread_pool(self, pool_name: str, max_workers: int = 4):
        """获取线程池"""
        if pool_name not in self.thread_pools:
            from concurrent.futures import ThreadPoolExecutor
            self.thread_pools[pool_name] = ThreadPoolExecutor(max_workers=max_workers)
        return self.thread_pools[pool_name]

    def cleanup(self):
        """清理资源"""
        # 关闭线程池
        for pool in self.thread_pools.values():
            pool.shutdown(wait=False)

        # 清理内存池
        self.memory_pools.clear()

        print("Resource cleanup completed")

# 使用资源管理器
class ResourceOptimizedAgent:
    """资源优化的智能体"""

    def __init__(self, agent_id: str, system_message: str, resource_manager: ResourceManager):
        self.agent_id = agent_id
        self.resource_manager = resource_manager

        # 创建智能体创建函数
        def create_agent():
            return ChatAgent(
                system_message=system_message,
                model=ModelFactory.create("openai", "gpt-4o-mini")
            )

        self.create_func = create_agent

    async def process_with_resources(self, message: str) -> Any:
        """使用资源优化的方式处理消息"""
        # 使用连接池获取智能体
        with self.resource_manager.get_agent_connection(self.agent_id, self.create_func) as agent:
            # 使用内存池
            memory_chunk = self.resource_manager.get_memory_chunk("agent_memory")

            try:
                # 异步处理消息
                response = await agent.astep(message)

                # 使用线程池进行后处理
                thread_pool = self.resource_manager.get_thread_pool("post_processing")

                def post_process(response_text):
                    # 模拟后处理
                    return f"Processed: {response_text[:100]}..."

                loop = asyncio.get_event_loop()
                processed_result = await loop.run_in_executor(
                    thread_pool, post_process, response.msgs[0].content
                )

                return {
                    "original_response": response.msgs[0].content,
                    "processed_result": processed_result,
                    "processing_time": response.info.get("response_time", 0)
                }

            finally:
                # 归还内存
                if memory_chunk:
                    self.resource_manager.return_memory_chunk("agent_memory", memory_chunk)

# 演示资源优化
async def demonstrate_resource_optimization():
    """演示资源优化效果"""
    resource_manager = ResourceManager()

    # 创建内存池
    resource_manager.create_memory_pool("agent_memory", pool_size=50)

    # 创建多个资源优化的智能体
    agents = []
    for i in range(3):
        agent = ResourceOptimizedAgent(
            f"agent_{i}",
            f"You are helpful agent {i}.",
            resource_manager
        )
        agents.append(agent)

    # 并行处理多个请求
    tasks = []
    for i in range(10):
        agent_idx = i % len(agents)
        task = agents[agent_idx].process_with_resources(f"Task {i}: What is {i + 1} + {i + 2}?")
        tasks.append(task)

    print("Starting resource-optimized processing...")
    start_time = time.time()

    results = await asyncio.gather(*tasks)

    end_time = time.time()
    total_time = end_time - start_time

    print(f"Processed {len(tasks)} tasks in {total_time:.2f} seconds")
    print(f"Average time per task: {total_time/len(tasks):.2f} seconds")

    # 显示一些结果
    for i, result in enumerate(results[:3]):
        print(f"\nResult {i+1}:")
        print(f"Processing time: {result['processing_time']:.2f}s")
        print(f"Processed: {result['processed_result']}")

    # 清理资源
    resource_manager.cleanup()

# 运行资源优化演示
asyncio.run(demonstrate_resource_optimization())
```

---

## 10. 错误处理和调试

### 10.1 全面的错误处理策略

```python
from camel.models import RateLimitError, ModelProcessingError, ModelBackendError
from camel.agents import ToolExecutionError
import logging
import traceback
from typing import Optional, Dict, Any
from dataclasses import dataclass

@dataclass
class ErrorContext:
    """错误上下文信息"""
    error_type: str
    error_message: str
    timestamp: float
    agent_id: Optional[str] = None
    task_id: Optional[str] = None
    user_request: Optional[str] = None
    stack_trace: Optional[str] = None

class AgentErrorHandler:
    """智能体错误处理器"""

    def __init__(self):
        self.error_log = []
        self.error_counts = {}
        self.recovery_strategies = {
            "RateLimitError": self._handle_rate_limit,
            "ModelProcessingError": self._handle_model_error,
            "ToolExecutionError": self._handle_tool_error,
            "TimeoutError": self._handle_timeout,
            "ConnectionError": self._handle_connection_error
        }

        # 配置日志
        logging.basicConfig(level=logging.INFO)
        self.logger = logging.getLogger(__name__)

    def handle_error(self, error: Exception, context: Dict[str, Any]) -> Dict[str, Any]:
        """处理错误"""
        error_type = type(error).__name__

        # 记录错误
        error_context = ErrorContext(
            error_type=error_type,
            error_message=str(error),
            timestamp=time.time(),
            stack_trace=traceback.format_exc(),
            **context
        )

        self._log_error(error_context)

        # 更新错误计数
        self.error_counts[error_type] = self.error_counts.get(error_type, 0) + 1

        # 应用恢复策略
        recovery_result = self._apply_recovery_strategy(error, error_context)

        return {
            "error_handled": True,
            "recovery_action": recovery_result["action"],
            "should_retry": recovery_result["retry"],
            "delay_seconds": recovery_result.get("delay", 0),
            "fallback_response": recovery_result.get("fallback")
        }

    def _log_error(self, context: ErrorContext):
        """记录错误"""
        self.error_log.append(context)

        # 按严重程度记录
        if context.error_type in ["RateLimitError", "ModelBackendError"]:
            self.logger.error(f"Critical error: {context.error_type} - {context.error_message}")
        else:
            self.logger.warning(f"Handled error: {context.error_type} - {context.error_message}")

    def _apply_recovery_strategy(self, error: Exception, context: ErrorContext) -> Dict[str, Any]:
        """应用恢复策略"""
        error_type = type(error).__name__

        if error_type in self.recovery_strategies:
            return self.recovery_strategies[error_type](error, context)
        else:
            return self._handle_generic_error(error, context)

    def _handle_rate_limit(self, error: Exception, context: ErrorContext) -> Dict[str, Any]:
        """处理速率限制错误"""
        return {
            "action": "wait_and_retry",
            "retry": True,
            "delay": 60,  # 等待60秒
            "fallback": "I'm experiencing high demand right now. Please try again in a minute."
        }

    def _handle_model_error(self, error: Exception, context: ErrorContext) -> Dict[str, Any]:
        """处理模型错误"""
        return {
            "action": "switch_model",
            "retry": True,
            "fallback": "I'm having trouble with my current model. Let me try a different approach."
        }

    def _handle_tool_error(self, error: Exception, context: ErrorContext) -> Dict[str, Any]:
        """处理工具错误"""
        return {
            "action": "disable_tool",
            "retry": True,
            "fallback": "I'm unable to use that tool right now. Let me try to help you in a different way."
        }

    def _handle_timeout(self, error: Exception, context: ErrorContext) -> Dict[str, Any]:
        """处理超时错误"""
        return {
            "action": "reduce_complexity",
            "retry": True,
            "fallback": "That request is taking too long. Let me try a simpler approach."
        }

    def _handle_connection_error(self, error: Exception, context: ErrorContext) -> Dict[str, Any]:
        """处理连接错误"""
        return {
            "action": "retry_with_backoff",
            "retry": True,
            "delay": 5,
            "fallback": "I'm having trouble connecting right now. Please try again in a moment."
        }

    def _handle_generic_error(self, error: Exception, context: ErrorContext) -> Dict[str, Any]:
        """处理通用错误"""
        return {
            "action": "log_and_continue",
            "retry": False,
            "fallback": "I encountered an unexpected error. Let me try to help you with what I can do."
        }

    def get_error_stats(self) -> Dict[str, Any]:
        """获取错误统计"""
        total_errors = sum(self.error_counts.values())

        return {
            "total_errors": total_errors,
            "error_distribution": self.error_counts.copy(),
            "most_common_error": max(self.error_counts.items(), key=lambda x: x[1])[0] if self.error_counts else None,
            "recent_errors": len([e for e in self.error_log if time.time() - e.timestamp < 3600])  # 最近1小时
        }

class ResilientChatAgent:
    """弹性聊天智能体"""

    def __init__(self, system_message: str, model_configs: list):
        self.system_message = system_message
        self.model_configs = model_configs
        self.current_model_index = 0
        self.error_handler = AgentErrorHandler()

        # 创建模型
        self.models = []
        for config in model_configs:
            try:
                model = ModelFactory.create(**config)
                self.models.append(model)
            except Exception as e:
                self.error_handler.handle_error(e, {"action": "model_creation"})

    def step_with_retry(self, message: str, max_retries: int = 3) -> Any:
        """带重试的智能体调用"""
        last_error = None

        for attempt in range(max_retries):
            try:
                # 获取当前模型
                if not self.models:
                    raise Exception("No available models")

                current_model = self.models[self.current_model_index]

                # 创建临时智能体
                agent = ChatAgent(
                    system_message=self.system_message,
                    model=current_model,
                    step_timeout=30.0,
                    retry_attempts=1  # 在这里控制重试，避免无限重试
                )

                # 执行调用
                response = agent.step(message)

                # 成功，重置模型索引
                self.current_model_index = 0
                return response

            except Exception as e:
                last_error = e

                # 处理错误
                context = {
                    "agent_id": id(self),
                    "user_request": message[:100],
                    "attempt": attempt + 1,
                    "current_model": self.models[self.current_model_index].model_type if self.models else "none"
                }

                error_result = self.error_handler.handle_error(e, context)

                # 根据错误处理结果决定下一步
                if error_result["should_retry"]:
                    if error_result.get("delay", 0) > 0:
                        time.sleep(error_result["delay"])

                    if error_result["action"] == "switch_model" and len(self.models) > 1:
                        # 切换到下一个模型
                        self.current_model_index = (self.current_model_index + 1) % len(self.models)
                        print(f"Switched to model {self.models[self.current_model_index].model_type}")

                    continue
                else:
                    # 不重试，返回备用响应
                    if error_result.get("fallback"):
                        return self._create_fallback_response(error_result["fallback"])
                    break

        # 所有重试都失败
        return self._create_error_response(last_error)

    def _create_fallback_response(self, message: str) -> Any:
        """创建备用响应"""
        # 创建一个模拟的响应对象
        class FallbackResponse:
            def __init__(self, msg):
                self.msgs = [msg]
                self.terminated = True
                self.info = {"fallback": True}

        return FallbackResponse(BaseMessage.make_assistant_message(
            role_name="Assistant",
            content=message
        ))

    def _create_error_response(self, error: Exception) -> Any:
        """创建错误响应"""
        error_message = f"I apologize, but I'm experiencing technical difficulties: {str(error)}"
        return self._create_fallback_response(error_message)

# 使用弹性智能体
def demonstrate_resilient_agent():
    """演示弹性智能体"""
    # 配置多个模型作为备用
    model_configs = [
        {"model_platform": "openai", "model_type": "gpt-4o-mini"},
        {"model_platform": "anthropic", "model_type": "claude-3-5-sonnet"},
        {"model_platform": "google", "model_type": "gemini-pro"}
    ]

    resilient_agent = ResilientChatAgent(
        system_message="You are a helpful assistant with error resilience.",
        model_configs=model_configs
    )

    # 测试各种场景
    test_messages = [
        "What is 2+2?",
        "Explain quantum computing in simple terms",
        "Write a Python function for fibonacci sequence"
    ]

    print("Testing resilient agent...")

    for message in test_messages:
        print(f"\nTesting: {message}")
        try:
            response = resilient_agent.step_with_retry(message, max_retries=3)
            print(f"Success: {response.msgs[0].content[:100]}...")
        except Exception as e:
            print(f"Failed after all retries: {e}")

        # 显示错误统计
        error_stats = resilient_agent.error_handler.get_error_stats()
        print(f"Error stats: {error_stats['total_errors']} total errors")

# 运行弹性智能体演示
demonstrate_resilient_agent()
```

### 10.2 调试和监控工具

```python
import time
import psutil
import threading
from typing import Dict, Any, List, Callable
from dataclasses import dataclass, field
from collections import deque

@dataclass
class PerformanceMetric:
    """性能指标"""
    timestamp: float
    metric_name: str
    value: float
    metadata: Dict[str, Any] = field(default_factory=dict)

class AgentMonitor:
    """智能体监控器"""

    def __init__(self, max_metrics: int = 1000):
        self.max_metrics = max_metrics
        self.metrics: Dict[str, deque] = {}
        self.alerts: List[Dict[str, Any]] = []
        self.thresholds: Dict[str, Dict[str, float]] = {}
        self.callbacks: Dict[str, List[Callable]] = {}

        # 启动监控线程
        self.monitoring_thread = threading.Thread(target=self._monitor_resources, daemon=True)
        self.monitoring_thread.start()

    def add_metric(self, metric_name: str, value: float, metadata: Dict[str, Any] = None):
        """添加性能指标"""
        if metric_name not in self.metrics:
            self.metrics[metric_name] = deque(maxlen=self.max_metrics)

        metric = PerformanceMetric(
            timestamp=time.time(),
            metric_name=metric_name,
            value=value,
            metadata=metadata or {}
        )

        self.metrics[metric_name].append(metric)

        # 检查阈值
        self._check_thresholds(metric)

    def set_threshold(self, metric_name: str, threshold_type: str, value: float, callback: Callable = None):
        """设置阈值"""
        if metric_name not in self.thresholds:
            self.thresholds[metric_name] = {}

        self.thresholds[metric_name][threshold_type] = value

        if callback:
            if metric_name not in self.callbacks:
                self.callbacks[metric_name] = []
            self.callbacks[metric_name].append(callback)

    def _check_thresholds(self, metric: PerformanceMetric):
        """检查阈值"""
        if metric.metric_name not in self.thresholds:
            return

        thresholds = self.thresholds[metric.metric_name]

        # 检查各种阈值
        for threshold_type, threshold_value in thresholds.items():
            triggered = False

            if threshold_type == "max" and metric.value > threshold_value:
                triggered = True
            elif threshold_type == "min" and metric.value < threshold_value:
                triggered = True
            elif threshold_type == "spike" and len(self.metrics[metric.metric_name]) > 1:
                prev_value = self.metrics[metric.metric_name][-2].value
                if abs(metric.value - prev_value) > threshold_value:
                    triggered = True

            if triggered:
                alert = {
                    "metric_name": metric.metric_name,
                    "threshold_type": threshold_type,
                    "threshold_value": threshold_value,
                    "actual_value": metric.value,
                    "timestamp": metric.timestamp,
                    "metadata": metric.metadata
                }

                self.alerts.append(alert)
                print(f"ALERT: {alert}")

                # 调用回调函数
                if metric.metric_name in self.callbacks:
                    for callback in self.callbacks[metric.metric_name]:
                        try:
                            callback(alert)
                        except Exception as e:
                            print(f"Error in threshold callback: {e}")

    def _monitor_resources(self):
        """监控系统资源"""
        while True:
            try:
                # CPU使用率
                cpu_percent = psutil.cpu_percent()
                self.add_metric("cpu_usage", cpu_percent)

                # 内存使用率
                memory = psutil.virtual_memory()
                self.add_metric("memory_usage", memory.percent)

                # 磁盘使用率
                disk = psutil.disk_usage('/')
                self.add_metric("disk_usage", disk.percent)

                time.sleep(5)  # 每5秒采集一次

            except Exception as e:
                print(f"Error in resource monitoring: {e}")
                time.sleep(10)

    def get_metric_summary(self, metric_name: str, time_range: int = 300) -> Dict[str, Any]:
        """获取指标摘要"""
        if metric_name not in self.metrics:
            return {}

        current_time = time.time()
        recent_metrics = [
            m for m in self.metrics[metric_name]
            if current_time - m.timestamp <= time_range
        ]

        if not recent_metrics:
            return {}

        values = [m.value for m in recent_metrics]

        return {
            "metric_name": metric_name,
            "count": len(values),
            "min": min(values),
            "max": max(values),
            "avg": sum(values) / len(values),
            "latest": values[-1] if values else None,
            "trend": self._calculate_trend(values)
        }

    def _calculate_trend(self, values: List[float]) -> str:
        """计算趋势"""
        if len(values) < 2:
            return "stable"

        first_half = values[:len(values)//2]
        second_half = values[len(values)//2:]

        first_avg = sum(first_half) / len(first_half)
        second_avg = sum(second_half) / len(second_half)

        if second_avg > first_avg * 1.1:
            return "increasing"
        elif second_avg < first_avg * 0.9:
            return "decreasing"
        else:
            return "stable"

class DebugAgent:
    """调试智能体"""

    def __init__(self, base_agent: ChatAgent, monitor: AgentMonitor):
        self.base_agent = base_agent
        self.monitor = monitor
        self.request_count = 0
        self.debug_mode = False

    def step(self, message: str) -> Any:
        """带调试的智能体调用"""
        self.request_count += 1
        start_time = time.time()

        try:
            # 记录请求开始
            self.monitor.add_metric(
                "request_started",
                self.request_count,
                {"message_length": len(message)}
            )

            # 执行智能体调用
            response = self.base_agent.step(message)

            # 记录成功指标
            end_time = time.time()
            response_time = end_time - start_time

            self.monitor.add_metric(
                "response_time",
                response_time,
                {"request_id": self.request_count, "success": True}
            )

            # 记录token使用情况
            if hasattr(response, 'info') and 'usage' in response.info:
                usage = response.info['usage']
                self.monitor.add_metric(
                    "tokens_used",
                    usage.get('total_tokens', 0),
                    {"request_id": self.request_count}
                )

            # 调试输出
            if self.debug_mode:
                self._debug_output(message, response, response_time)

            return response

        except Exception as e:
            # 记录错误指标
            end_time = time.time()
            response_time = end_time - start_time

            self.monitor.add_metric(
                "response_time",
                response_time,
                {"request_id": self.request_count, "success": False, "error": str(e)}
            )

            self.monitor.add_metric(
                "error_count",
                1,
                {"request_id": self.request_count, "error_type": type(e).__name__}
            )

            if self.debug_mode:
                self._debug_error(message, e, response_time)

            raise

    def set_debug_mode(self, enabled: bool):
        """设置调试模式"""
        self.debug_mode = enabled

    def _debug_output(self, message: str, response: Any, response_time: float):
        """调试输出"""
        print(f"\n=== DEBUG OUTPUT ===")
        print(f"Request #{self.request_count}")
        print(f"Message: {message[:100]}...")
        print(f"Response time: {response_time:.3f}s")
        print(f"Response length: {len(str(response.msgs[0].content))} chars")

        if hasattr(response, 'info') and 'usage' in response.info:
            usage = response.info['usage']
            print(f"Token usage: {usage}")

        print("=" * 50)

    def _debug_error(self, message: str, error: Exception, response_time: float):
        """调试错误输出"""
        print(f"\n=== DEBUG ERROR ===")
        print(f"Request #{self.request_count}")
        print(f"Message: {message[:100]}...")
        print(f"Response time: {response_time:.3f}s")
        print(f"Error: {type(error).__name__}: {error}")
        print("=" * 50)

def performance_alert_callback(alert: Dict[str, Any]):
    """性能告警回调"""
    print(f"\n🚨 PERFORMANCE ALERT:")
    print(f"  Metric: {alert['metric_name']}")
    print(f"  Type: {alert['threshold_type']}")
    print(f"  Threshold: {alert['threshold_value']}")
    print(f"  Actual: {alert['actual_value']}")
    print(f"  Time: {time.strftime('%Y-%m-%d %H:%M:%S', time.localtime(alert['timestamp']))}")

def demonstrate_monitoring():
    """演示监控功能"""
    # 创建监控器
    monitor = AgentMonitor()

    # 设置阈值
    monitor.set_threshold("response_time", "max", 10.0, performance_alert_callback)
    monitor.set_threshold("error_count", "max", 0.1, performance_alert_callback)  # 10%错误率

    # 创建基础智能体
    base_agent = ChatAgent(
        system_message="You are a helpful assistant.",
        model=ModelFactory.create("openai", "gpt-4o-mini")
    )

    # 创建调试智能体
    debug_agent = DebugAgent(base_agent, monitor)
    debug_agent.set_debug_mode(True)

    # 测试监控
    print("Starting monitoring demonstration...")

    test_messages = [
        "What is the capital of France?",
        "Explain the theory of relativity in simple terms",
        "Write a Python function to calculate factorial",
        "What are the main causes of climate change?",
        "How does blockchain technology work?"
    ]

    for message in test_messages:
        try:
            response = debug_agent.step(message)
            print(f"✓ Completed: {message[:30]}...")
        except Exception as e:
            print(f"✗ Failed: {message[:30]}... - {e}")

        time.sleep(1)  # 模拟真实间隔

    # 显示监控摘要
    print("\n=== MONITORING SUMMARY ===")

    for metric_name in ["response_time", "tokens_used", "error_count"]:
        summary = monitor.get_metric_summary(metric_name)
        if summary:
            print(f"\n{metric_name.replace('_', ' ').title()}:")
            print(f"  Count: {summary['count']}")
            print(f"  Average: {summary['avg']:.3f}")
            print(f"  Min: {summary['min']:.3f}")
            print(f"  Max: {summary['max']:.3f}")
            print(f"  Trend: {summary['trend']}")

    # 显示告警
    if monitor.alerts:
        print(f"\nTotal alerts: {len(monitor.alerts)}")
        for alert in monitor.alerts[-3:]:  # 显示最近3个告警
            print(f"  {alert['metric_name']} {alert['threshold_type']}: {alert['actual_value']:.2f}")

# 运行监控演示
demonstrate_monitoring()
```

---

## 总结

本文档详细介绍了CAMEL框架的使用示例和最佳实践，涵盖了：

### 主要特点：

1. **快速入门**：简单的环境配置和基础使用
2. **智能体使用**：多种初始化方式和配置选项
3. **多智能体协作**：角色扮演、工作力系统、BabyAGI等协作模式
4. **工具集成**：内置工具包和自定义工具开发
5. **记忆系统**：多种记忆类型和上下文管理策略
6. **模型管理**：负载均衡、成本优化、智能选择
7. **任务管理**：工作流、动态任务生成、迭代改进
8. **高级应用**：多语言系统、代码审查、智能客服
9. **性能优化**：异步处理、缓存机制、资源管理
10. **错误处理**：全面的错误处理策略和监控工具

### 最佳实践要点：

- **配置优化**：根据应用场景选择合适的配置参数
- **错误处理**：实现健壮的错误处理和恢复机制
- **资源管理**：合理使用连接池、内存池等资源管理技术
- **性能监控**：建立完善的监控和告警系统
- **模块化设计**：将复杂功能拆分为可重用的模块
- **测试验证**：充分测试各种场景和边界情况

### 适用场景：

- **研究应用**：多智能体协作研究、任务自动化
- **商业应用**：客服系统、内容生成、数据分析
- **开发应用**：代码生成、测试自动化、文档生成
- **教育应用**：个性化学习、智能辅导

通过这些示例和最佳实践，开发者可以充分发挥CAMEL框架的潜力，构建功能强大、性能优异的多智能体应用。