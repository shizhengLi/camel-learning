# CAMEL框架API参考文档

## 1. 概述

CAMEL框架提供了完整的多智能体系统API接口，支持智能体创建、消息传递、工具调用、记忆管理、模型调度等核心功能。本文档详细介绍了框架的主要API接口及其使用方法。

## 2. 核心模块API

### 2.1 Agents模块

#### BaseAgent（抽象基类）

**类路径**: `camel.agents.base.BaseAgent`

**描述**: 所有智能体的抽象基类，定义了智能体的基本接口

**核心方法**:

```python
@abstractmethod
def reset(self, *args, **kwargs) -> Any:
    """重置智能体到初始状态"""
    pass

@abstractmethod
def step(self, *args, **kwargs) -> Any:
    """执行智能体的单步操作"""
    pass
```

#### ChatAgent（核心聊天智能体）

**类路径**: `camel.agents.chat_agent.ChatAgent`

**描述**: 框架的核心智能体类，支持多模型、工具调用、记忆管理等功能

**构造函数**:

```python
def __init__(
    self,
    system_message: Optional[Union[BaseMessage, str]] = None,
    model: Optional[Union[BaseModelBackend, ModelManager, Tuple[str, str], str,
                      ModelType, List[BaseModelBackend], List[str]]] = None,
    memory: Optional[AgentMemory] = None,
    message_window_size: Optional[int] = None,
    token_limit: Optional[int] = None,
    output_language: Optional[str] = None,
    tools: Optional[List[Union[FunctionTool, Callable]]] = None,
    toolkits_to_register_agent: Optional[List[RegisteredAgentToolkit]] = None,
    external_tools: Optional[List[Union[FunctionTool, Callable, Dict[str, Any]]]] = None,
    response_terminators: Optional[List[ResponseTerminator]] = None,
    scheduling_strategy: str = "round_robin",
    max_iteration: Optional[int] = None,
    agent_id: Optional[str] = None,
    stop_event: Optional[threading.Event] = None,
    tool_execution_timeout: Optional[float] = None,
    mask_tool_output: bool = False,
    pause_event: Optional[asyncio.Event] = None,
    prune_tool_calls_from_memory: bool = False,
    retry_attempts: int = 3,
    retry_delay: float = 1.0,
    step_timeout: Optional[float] = None,
    stream_accumulate: bool = True,
) -> None
```

**参数说明**:

| 参数名 | 类型 | 默认值 | 描述 |
|--------|------|--------|------|
| `system_message` | `Union[BaseMessage, str]` | `None` | 系统消息，定义智能体的角色和行为 |
| `model` | `Union[BaseModelBackend, ModelManager, ...]` | `None` | 使用的模型，支持多种格式 |
| `memory` | `Optional[AgentMemory]` | `None` | 记忆系统，用于保存对话历史 |
| `token_limit` | `Optional[int]` | `None` | Token限制，控制上下文长度 |
| `tools` | `Optional[List[Union[FunctionTool, Callable]]]` | `None` | 工具列表，扩展智能体功能 |
| `response_terminators` | `Optional[List[ResponseTerminator]]` | `None` | 响应终止器列表 |
| `scheduling_strategy` | `str` | `"round_robin"` | 模型调度策略 |
| `retry_attempts` | `int` | `3` | 重试次数 |
| `retry_delay` | `float` | `1.0` | 重试延迟（秒） |
| `step_timeout` | `Optional[float]` | `None` | 单步超时时间（秒） |

**核心方法**:

```python
def step(
    self,
    input_message: Union[BaseMessage, str],
    response_format: Optional[Type[BaseModel]] = None,
) -> Union[ChatAgentResponse, StreamingChatAgentResponse]:
    """执行单步聊天会话

    Args:
        input_message: 输入消息
        response_format: 响应格式类型

    Returns:
        ChatAgentResponse: 聊天响应对象
    """

def reset(self) -> None:
    """重置智能体状态"""

async def astep(
    self,
    input_message: Union[BaseMessage, str],
    response_format: Optional[Type[BaseModel]] = None,
) -> AsyncChatAgentResponse:
    """异步执行单步聊天会话"""
```

**使用示例**:

```python
# 基础使用
agent = ChatAgent("You are a helpful assistant.", model="gpt-4o-mini")
response = agent.step("Hello, how are you?")
print(response.msgs[0].content)

# 完整配置
from camel.models import ModelFactory
from camel.toolkits import SearchToolkit
from camel.memories import ChatHistoryMemory

model = ModelFactory.create(
    model_platform="openai",
    model_type="gpt-4o-mini",
)

agent = ChatAgent(
    system_message="You are a helpful assistant that can search the web.",
    model=model,
    tools=[FunctionTool(SearchToolkit().search_brave)],
    memory=ChatHistoryMemory(window_size=100),
    token_limit=4000,
    retry_attempts=3,
    step_timeout=30.0
)

response = agent.step("What's the latest news about AI?")
```

#### 其他智能体类型

**TaskSpecifyAgent**（任务规范智能体）
- `类路径`: `camel.agents.task_agents.TaskSpecifyAgent`
- `功能`: 规范化和明确用户提出的任务

**TaskPlannerAgent**（任务规划智能体）
- `类路径`: `camel.agents.task_agents.TaskPlannerAgent`
- `功能`: 将复杂任务分解为可执行的子任务

**CriticAgent**（评估智能体）
- `类路径`: `camel.agents.critic_agent.CriticAgent`
- `功能`: 评估和评论其他智能体的输出

**SearchAgent**（搜索智能体）
- `类路径`: `camel.agents.search_agent.SearchAgent`
- `功能`: 专门用于信息搜索的智能体

### 2.2 Messages模块

#### BaseMessage（基础消息类）

**类路径**: `camel.messages.base.BaseMessage`

**描述**: 框架的基础消息类，支持多模态内容

**构造函数**:

```python
@dataclass
class BaseMessage:
    role_name: str                    # 角色名称
    role_type: RoleType              # 角色类型
    meta_dict: Optional[Dict[str, Any]]  # 元数据字典
    content: str                      # 消息内容
    video_bytes: Optional[bytes] = None    # 视频数据
    image_list: Optional[List[Image.Image]] = None  # 图像列表
    image_detail: Literal["auto", "low", "high"] = "auto"  # 图像细节级别
    video_detail: Literal["auto", "low", "high"] = "auto"  # 视频细节级别
    parsed: Optional[Union[BaseModel, dict]] = None  # 解析结果
```

**类方法**:

```python
@classmethod
def make_user_message(
    cls,
    role_name: str,
    content: str,
    meta_dict: Optional[Dict[str, str]] = None,
    video_bytes: Optional[bytes] = None,
    image_list: Optional[List[Image.Image]] = None,
    image_detail: Union[OpenAIVisionDetailType, str] = OpenAIVisionDetailType.AUTO,
    video_detail: Union[OpenAIVisionDetailType, str] = OpenAIVisionDetailType.AUTO,
) -> "BaseMessage":
    """创建用户消息"""

@classmethod
def make_assistant_message(
    cls,
    role_name: str,
    content: str,
    meta_dict: Optional[Dict[str, str]] = None,
    video_bytes: Optional[bytes] = None,
    image_list: Optional[List[Image.Image]] = None,
    image_detail: Union[OpenAIVisionDetailType, str] = OpenAIVisionDetailType.AUTO,
    video_detail: Union[OpenAIVisionDetailType, str] = OpenAIVisionDetailType.AUTO,
) -> "BaseMessage":
    """创建助手消息"""

@classmethod
def make_system_message(
    cls,
    role_name: str,
    content: str,
    meta_dict: Optional[Dict[str, str]] = None,
) -> "BaseMessage":
    """创建系统消息"""
```

**使用示例**:

```python
# 创建不同类型的消息
user_msg = BaseMessage.make_user_message(
    role_name="User",
    content="Hello, how are you?"
)

assistant_msg = BaseMessage.make_assistant_message(
    role_name="Assistant",
    content="I'm doing well, thank you!"
)

system_msg = BaseMessage.make_system_message(
    role_name="System",
    content="You are a helpful assistant."
)

# 创建多模态消息
from PIL import Image

image = Image.open("example.jpg")
multimodal_msg = BaseMessage.make_user_message(
    role_name="User",
    content="What do you see in this image?",
    image_list=[image],
    image_detail="high"
)
```

#### FunctionCallingMessage（函数调用消息）

**类路径**: `camel.messages.func_message.FunctionCallingMessage`

**描述**: 专门用于函数调用的消息类型

**构造函数**:

```python
def __init__(
    self,
    func_name: str,
    args: Dict[str, Any],
    result: Any,
    role_name: str = "function_call"
) -> None
```

### 2.3 Societies模块

#### RolePlaying（角色扮演）

**类路径**: `camel.societies.role_playing.RolePlaying`

**描述**: 实现两个智能体之间的角色扮演对话

**构造函数**:

```python
def __init__(
    self,
    assistant_role_name: str,
    user_role_name: str,
    critic_role_name: str = "critic",
    task_prompt: str = "",
    with_task_specify: bool = True,
    with_task_planner: bool = False,
    with_critic_in_the_loop: bool = False,
    critic_criteria: Optional[str] = None,
    model: Optional[BaseModelBackend] = None,
    task_type: TaskType = TaskType.AI_SOCIETY,
    assistant_agent_kwargs: Optional[Dict] = None,
    user_agent_kwargs: Optional[Dict] = None,
    critic_agent_kwargs: Optional[Dict] = None,
    task_specify_agent_kwargs: Optional[Dict] = None,
    task_planner_agent_kwargs: Optional[Dict] = None,
) -> None
```

**参数说明**:

| 参数名 | 类型 | 默认值 | 描述 |
|--------|------|--------|------|
| `assistant_role_name` | `str` | 必填 | 助手角色名称 |
| `user_role_name` | `str` | 必填 | 用户角色名称 |
| `task_prompt` | `str` | `""` | 任务提示 |
| `with_task_specify` | `bool` | `True` | 是否启用任务规范 |
| `with_task_planner` | `bool` | `False` | 是否启用任务规划 |
| `with_critic_in_the_loop` | `bool` | `False` | 是否启用评论家循环 |
| `model` | `Optional[BaseModelBackend]` | `None` | 使用的模型 |

**核心方法**:

```python
def step(self, input_message: BaseMessage) -> Tuple[ChatAgentResponse, ChatAgentResponse]:
    """执行单步角色扮演对话

    Returns:
        Tuple[ChatAgentResponse, ChatAgentResponse]: (助手响应, 用户响应)
    """

def init_chat(self) -> BaseMessage:
    """初始化聊天，返回初始消息"""

def reset(self) -> None:
    """重置角色扮演会话"""
```

**使用示例**:

```python
# 基础角色扮演
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

# 继续对话直到终止
while not (assistant_response.terminated or user_response.terminated):
    input_msg = assistant_response.msg
    assistant_response, user_response = role_play.step(input_msg)

# 带评论家的角色扮演
role_play_with_critic = RolePlaying(
    assistant_role_name="Writer",
    user_role_name="Editor",
    task_prompt="Write a short story about AI",
    with_critic_in_the_loop=True,
    critic_criteria="Evaluate the story for creativity and coherence"
)
```

#### Workforce（工作力系统）

**类路径**: `camel.societies.workforce.Workforce`

**描述**: 管理多个工作者的任务执行系统

**核心方法**:

```python
def run(self) -> WorkforceRecord:
    """执行工作力任务

    Returns:
        WorkforceRecord: 工作力执行记录
    """
```

#### BabyAGI（自主任务生成系统）

**类路径**: `camel.societies.babyagi.BabyAGI`

**描述**: 自主生成和管理任务的智能体系统

**核心方法**:

```python
def run(self) -> BabyAGIRecord:
    """运行BabyAGI系统

    Returns:
        BabyAGIRecord: BabyAGI执行记录
    """
```

### 2.4 Toolkits模块

#### BaseToolkit（基础工具包）

**类路径**: `camel.toolkits.base.BaseToolkit`

**描述**: 所有工具包的抽象基类

**构造函数**:

```python
def __init__(self, timeout: Optional[float] = None) -> None:
    """初始化工具包

    Args:
        timeout: 工具执行超时时间（秒）
    """
```

**核心方法**:

```python
def get_tools(self) -> List[FunctionTool]:
    """获取工具列表

    Returns:
        List[FunctionTool]: 工具列表
    """
    raise NotImplementedError("Subclasses must implement this method.")
```

#### FunctionTool（函数工具）

**类路径**: `camel.toolkits.function_tool.FunctionTool`

**描述**: 将函数封装为智能体可调用的工具

**构造函数**:

```python
def __init__(
    self,
    func: Callable,
    openai_tool_schema: Optional[Dict[str, Any]] = None,
    synthesize_schema: Optional[bool] = False,
    synthesize_schema_model: Optional[BaseModelBackend] = None,
    synthesize_schema_max_retries: int = 2,
    synthesize_output: Optional[bool] = False,
    synthesize_output_model: Optional[BaseModelBackend] = None,
    synthesize_output_format: Optional[Type[BaseModel]] = None,
) -> None
```

**核心属性**:

```python
@property
def name(self) -> str:
    """工具名称"""

@property
def description(self) -> str:
    """工具描述"""

@property
def parameters(self) -> Dict[str, Any]:
    """工具参数模式"""
```

**使用示例**:

```python
# 创建自定义工具
def calculate_sum(a: int, b: int) -> int:
    """计算两个数的和"""
    return a + b

sum_tool = FunctionTool(calculate_sum)

# 与ChatAgent结合使用
agent = ChatAgent(
    system_message="You are a helpful assistant that can perform calculations.",
    tools=[sum_tool]
)

response = agent.step("What is the sum of 15 and 27?")
# 智能体会自动调用calculate_sum函数
```

#### SearchToolkit（搜索工具包）

**类路径**: `camel.toolkits.search_toolkit.SearchToolkit`

**描述**: 提供多种搜索引擎功能的工具包

**核心方法**:

```python
def search_brave(self, q: str, search_lang: str = "en") -> Dict[str, Any]:
    """使用Brave搜索引擎

    Args:
        q: 搜索查询
        search_lang: 搜索语言

    Returns:
        Dict[str, Any]: 搜索结果
    """

def search_baidu(self, query: str) -> Dict[str, Any]:
    """使用百度搜索引擎"""

def search_bing(self, query: str) -> Dict[str, Any]:
    """使用Bing搜索引擎"""

def search_exa(self, query: str, category: str = "github", num_results: int = 5) -> Dict[str, Any]:
    """使用Exa搜索引擎"""
```

**使用示例**:

```python
# 直接使用工具包
search_toolkit = SearchToolkit()
result = search_toolkit.search_brave(q="What is the weather in Tokyo?")

# 与ChatAgent结合使用
agent = ChatAgent(
    system_message="You are a helpful assistant that can search the web.",
    tools=[FunctionTool(search_toolkit.search_brave)]
)

response = agent.step("What's the latest news about artificial intelligence?")
```

#### 其他工具包

**CodeExecutionToolkit**（代码执行工具包）
- `类路径`: `camel.toolkits.code_execution_toolkit.CodeExecutionToolkit`
- `功能`: 执行Python、Bash、SQL代码

**BrowserToolkit**（浏览器工具包）
- `类路径`: `camel.toolkits.browser_toolkit.BrowserToolkit`
- `功能`: 浏览器自动化操作

**MathToolkit**（数学工具包）
- `类路径`: `camel.toolkits.math_toolkit.MathToolkit`
- `功能`: 数学计算和符号运算

### 2.5 Memories模块

#### AgentMemory（智能体记忆接口）

**类路径**: `camel.memories.base.AgentMemory`

**描述**: 智能体记忆系统的抽象接口

**核心方法**:

```python
@abstractmethod
def get_records(self, num: Optional[int] = None) -> List[MemoryRecord]:
    """获取记忆记录"""

@abstractmethod
def add_record(self, record: MemoryRecord) -> None:
    """添加记忆记录"""

@abstractmethod
def clear(self) -> None:
    """清空记忆"""
```

#### ChatHistoryMemory（聊天历史记忆）

**类路径**: `camel.memories.agent_memories.ChatHistoryMemory`

**描述**: 基于窗口的聊天历史记忆实现

**构造函数**:

```python
def __init__(
    self,
    window_size: int = 1000,
    token_limit: int = 4000,
    enable_tool_calling_messages: bool = False
) -> None:
    """初始化聊天历史记忆

    Args:
        window_size: 记忆窗口大小
        token_limit: Token限制
        enable_tool_calling_messages: 是否启用工具调用消息
    """
```

**使用示例**:

```python
# 创建记忆系统
memory = ChatHistoryMemory(
    window_size=100,
    token_limit=4000,
    enable_tool_calling_messages=False
)

# 与ChatAgent结合使用
agent = ChatAgent(
    system_message="You are a helpful assistant.",
    memory=memory
)

# 对话会自动保存到记忆中
response1 = agent.step("Hello!")
response2 = agent.step("How are you?")

# 查看记忆记录
records = memory.get_records()
print(f"Total records: {len(records)}")
```

#### VectorDBMemory（向量数据库记忆）

**类路径**: `camel.memories.agent_memories.VectorDBMemory`

**描述**: 基于向量数据库的语义搜索记忆

**构造函数**:

```python
def __init__(
    self,
    embedding_model: str = "text-embedding-ada-002",
    collection_name: str = "agent_memory",
    similarity_threshold: float = 0.7,
    max_results: int = 10
) -> None:
```

**核心方法**:

```python
def semantic_search(
    self,
    query: str,
    max_results: Optional[int] = None
) -> List[MemoryRecord]:
    """语义搜索相关记忆"""
```

### 2.6 Models模块

#### BaseModelBackend（基础模型后端）

**类路径**: `camel.models.base_model.BaseModelBackend`

**描述**: 模型后端的抽象基类

**构造函数**:

```python
def __init__(
    self,
    model_type: Union[ModelType, str],
    model_config_dict: Optional[Dict[str, Any]] = None,
    api_key: Optional[str] = None,
    url: Optional[str] = None,
    token_counter: Optional[BaseTokenCounter] = None,
    timeout: Optional[float] = None,
    max_retries: int = 3,
) -> None
```

**核心方法**:

```python
@abstractmethod
def run(self, messages: List[Dict[str, Any]], **kwargs) -> Any:
    """运行模型"""

@abstractmethod
def run_stream(self, messages: List[Dict[str, Any]], **kwargs) -> Any:
    """运行流式模型"""
```

#### ModelFactory（模型工厂）

**类路径**: `camel.models.model_factory.ModelFactory`

**描述**: 创建和管理模型实例的工厂类

**核心方法**:

```python
@classmethod
def create(
    cls,
    model_platform: Union[ModelPlatformType, str],
    model_type: Union[ModelType, str],
    model_config_dict: Optional[Dict[str, Any]] = None,
    api_key: Optional[str] = None,
    url: Optional[str] = None,
    **kwargs
) -> BaseModelBackend:
    """创建模型实例

    Args:
        model_platform: 模型平台 (openai, anthropic, etc.)
        model_type: 模型类型
        model_config_dict: 模型配置字典
        api_key: API密钥
        url: 模型URL
        **kwargs: 其他参数

    Returns:
        BaseModelBackend: 模型实例
    """
```

**使用示例**:

```python
# 使用字符串创建模型
model = ModelFactory.create(
    model_platform="openai",
    model_type="gpt-4o-mini",
    model_config_dict={"temperature": 0.7}
)

# 使用枚举创建模型
from camel.types import ModelPlatformType, ModelType

model = ModelFactory.create(
    model_platform=ModelPlatformType.OPENAI,
    model_type=ModelType.GPT_4O_MINI
)
```

#### ModelManager（模型管理器）

**类路径**: `camel.models.model_manager.ModelManager`

**描述**: 管理多个模型实例，提供负载均衡和容错功能

**构造函数**:

```python
def __init__(
    self,
    models: Union[List[BaseModelBackend], List[str], BaseModelBackend, str],
    scheduling_strategy: str = "round_robin",
    **kwargs
) -> None:
    """初始化模型管理器

    Args:
        models: 模型列表或单个模型
        scheduling_strategy: 调度策略 ("round_robin", "random", "priority")
        **kwargs: 其他参数
    """
```

**支持的调度策略**:
- `"round_robin"`: 轮询调度
- `"random"`: 随机调度
- `"priority"`: 优先级调度
- `"always_first"`: 始终使用第一个模型

**使用示例**:

```python
# 创建多个模型
model1 = ModelFactory.create("openai", "gpt-4o-mini")
model2 = ModelFactory.create("anthropic", "claude-3-5-sonnet-latest")

# 创建模型管理器
model_manager = ModelManager(
    models=[model1, model2],
    scheduling_strategy="round_robin"
)

# 与ChatAgent结合使用
agent = ChatAgent(
    system_message="You are a helpful assistant.",
    model=model_manager
)
```

### 2.7 Tasks模块

#### Task（任务类）

**类路径**: `camel.tasks.task.Task`

**描述**: 表示任务的基类

**构造函数**:

```python
def __init__(
    self,
    content: str,
    id: str = "",
    parent_id: Optional[str] = None,
    children_ids: Optional[List[str]] = None,
    status: TaskStatus = TaskStatus.TODO,
    metadata: Optional[Dict[str, Any]] = None,
) -> None:
    """初始化任务

    Args:
        content: 任务内容
        id: 任务ID
        parent_id: 父任务ID
        children_ids: 子任务ID列表
        status: 任务状态
        metadata: 元数据
    """
```

**核心方法**:

```python
def decompose(self, agent: ChatAgent) -> List["Task"]:
    """分解任务为子任务

    Args:
        agent: 用于分解任务的智能体

    Returns:
        List[Task]: 子任务列表
    """

def evolve(self, agent: ChatAgent) -> Optional["Task"]:
    """演化任务

    Args:
        agent: 用于演化任务的智能体

    Returns:
        Optional[Task]: 演化后的任务，如果没有变化则返回None
    """

def to_string(self) -> str:
    """将任务转换为字符串表示"""
```

**使用示例**:

```python
# 创建任务
task = Task(
    content="Weng earns $12 an hour for babysitting. Yesterday, she just did 51 minutes of babysitting. How much did she earn?",
    id="math_problem",
    metadata={"difficulty": "easy", "subject": "math"}
)

# 分解任务
subtasks = task.decompose(agent=agent)

# 演化任务
evolved_task = task.evolve(agent=agent)
```

#### TaskManager（任务管理器）

**类路径**: `camel.tasks.task.TaskManager`

**描述**: 管理任务的分解、演化和执行

**核心方法**:

```python
def evolve(self, task: Task, agent: ChatAgent) -> Optional[Task]:
    """演化任务"""

def decompose(self, task: Task, agent: ChatAgent) -> List[Task]:
    """分解任务"""
```

## 3. 类型定义

### 3.1 模型类型

```python
class ModelPlatformType(Enum):
    OPENAI = "openai"
    ANTHROPIC = "anthropic"
    GOOGLE = "google"
    OLLAMA = "ollama"
    AZURE = "azure"
    BEDROCK = "bedrock"
    VLLM = "vllm"
    SGLANG = "sglang"

class ModelType(Enum):
    # OpenAI模型
    GPT_4O = "gpt-4o"
    GPT_4O_MINI = "gpt-4o-mini"
    GPT_4_TURBO = "gpt-4-turbo"
    GPT_3_5_TURBO = "gpt-3.5-turbo"

    # Anthropic模型
    CLAUDE_3_OPUS = "claude-3-opus"
    CLAUDE_3_SONNET = "claude-3-sonnet"
    CLAUDE_3_HAIKU = "claude-3-haiku"
    CLAUDE_3_5_SONNET_LATEST = "claude-3-5-sonnet-latest"
```

### 3.2 角色类型

```python
class RoleType(Enum):
    USER = "user"
    ASSISTANT = "assistant"
    SYSTEM = "system"
    FUNCTION = "function"
```

### 3.3 任务状态

```python
class TaskStatus(Enum):
    TODO = "todo"
    RUNNING = "running"
    DONE = "done"
    FAILED = "failed"
    CANCELLED = "cancelled"
```

## 4. 响应类型

### 4.1 ChatAgentResponse

**类路径**: `camel.responses.chat_agent_response.ChatAgentResponse`

**描述**: 聊天智能体的响应对象

**主要属性**:

```python
@property
def msgs(self) -> List[BaseMessage]:
    """响应消息列表"""

@property
def terminated(self) -> bool:
    """是否终止"""

@property
def info(self) -> Dict[str, Any]:
    """响应信息"""

@property
def session_id(self) -> Optional[str]:
    """会话ID"""

@property
def tool_calls(self) -> List[ToolCallingRecord]:
    """工具调用记录"""
```

### 4.2 StreamingChatAgentResponse

**类路径**: `camel.responses.chat_agent_response.StreamingChatAgentResponse`

**描述**: 流式聊天响应对象

**核心方法**:

```python
def __iter__(self) -> Iterator[ChatAgentResponse]:
    """迭代获取流式响应"""
```

## 5. 异常类型

### 5.1 模型异常

```python
class ModelBackendError(Exception):
    """模型后端错误"""

class ModelFactoryError(Exception):
    """模型工厂错误"""

class ModelManagerError(Exception):
    """模型管理器错误"""
```

### 5.2 智能体异常

```python
class AgentError(Exception):
    """智能体基础错误"""

class ChatAgentError(AgentError):
    """聊天智能体错误"""

class ToolExecutionError(AgentError):
    """工具执行错误"""
```

### 5.3 任务异常

```python
class TaskError(Exception):
    """任务基础错误"""

class TaskTimeoutError(TaskError):
    """任务超时错误"""

class TaskValidationError(TaskError):
    """任务验证错误"""
```

## 6. API使用最佳实践

### 6.1 智能体配置最佳实践

```python
# 1. 使用ModelFactory创建模型
model = ModelFactory.create(
    model_platform=ModelPlatformType.OPENAI,
    model_type=ModelType.GPT_4O_MINI,
    model_config_dict={
        "temperature": 0.7,
        "max_tokens": 1000
    }
)

# 2. 配置完整的ChatAgent
agent = ChatAgent(
    system_message="You are a helpful assistant.",
    model=model,
    memory=ChatHistoryMemory(
        window_size=100,
        token_limit=4000
    ),
    tools=[FunctionTool(custom_function)],
    response_terminators=[TokenLimitTerminator(2000)],
    retry_attempts=3,
    step_timeout=30.0
)
```

### 6.2 错误处理最佳实践

```python
from camel.models import RateLimitError, ModelProcessingError
from camel.agents import ToolExecutionError

def safe_agent_call(agent: ChatAgent, message: str) -> Optional[ChatAgentResponse]:
    """安全的智能体调用"""
    try:
        return agent.step(message)
    except RateLimitError as e:
        print(f"Rate limit exceeded: {e}")
        time.sleep(60)  # 等待后重试
        return None
    except ModelProcessingError as e:
        print(f"Model processing error: {e}")
        return None
    except ToolExecutionError as e:
        print(f"Tool execution error: {e}")
        return None
    except Exception as e:
        print(f"Unexpected error: {e}")
        return None
```

### 6.3 多智能体协作最佳实践

```python
# 1. 配置角色扮演
role_playing = RolePlaying(
    assistant_role_name="Python Developer",
    user_role_name="Code Reviewer",
    task_prompt="Develop a Python function for data processing",
    with_task_specify=True,
    with_critic_in_the_loop=True,
    model=model
)

# 2. 执行对话管理
def run_role_playing_session(role_playing: RolePlaying, max_turns: int = 10):
    """运行角色扮演会话"""
    input_msg = role_playing.init_chat()

    for turn in range(max_turns):
        try:
            assistant_response, user_response = role_playing.step(input_msg)

            if assistant_response.terminated or user_response.terminated:
                break

            print(f"Turn {turn + 1}:")
            print(f"Assistant: {assistant_response.msgs[0].content}")
            print(f"User: {user_response.msgs[0].content}")

            input_msg = assistant_response.msg

        except Exception as e:
            print(f"Error in turn {turn + 1}: {e}")
            break
```

### 6.4 工具开发最佳实践

```python
# 1. 创建自定义工具包
class CustomToolkit(BaseToolkit):
    def __init__(self):
        super().__init__()
        self.add_tools([
            FunctionTool(self.process_data),
            FunctionTool(self.analyze_results)
        ])

    def process_data(self, data: str) -> str:
        """处理数据"""
        try:
            # 数据处理逻辑
            return f"Processed: {data}"
        except Exception as e:
            return f"Error processing data: {e}"

    def analyze_results(self, results: str) -> str:
        """分析结果"""
        try:
            # 结果分析逻辑
            return f"Analysis: {results}"
        except Exception as e:
            return f"Error analyzing results: {e}"

# 2. 使用自定义工具包
custom_toolkit = CustomToolkit()
agent = ChatAgent(
    system_message="You are a data analysis assistant.",
    tools=custom_toolkit.get_tools()
)
```

## 7. 性能优化建议

### 7.1 模型管理优化

```python
# 1. 使用模型管理器实现负载均衡
model_manager = ModelManager(
    models=[
        ModelFactory.create("openai", "gpt-4o-mini"),
        ModelFactory.create("anthropic", "claude-3-5-sonnet-latest")
    ],
    scheduling_strategy="round_robin"
)

# 2. 配置合理的重试策略
agent = ChatAgent(
    model=model_manager,
    retry_attempts=3,
    retry_delay=1.0,
    step_timeout=30.0
)
```

### 7.2 记忆系统优化

```python
# 1. 配置合适的记忆窗口
memory = ChatHistoryMemory(
    window_size=100,  # 根据应用场景调整
    token_limit=4000,  # 根据模型上下文长度调整
    enable_tool_calling_messages=False  # 清理工具调用消息
)

# 2. 使用向量数据库进行语义搜索
vector_memory = VectorDBMemory(
    embedding_model="text-embedding-ada-002",
    similarity_threshold=0.7,
    max_results=10
)
```

### 7.3 异步处理优化

```python
# 1. 使用异步API
import asyncio

async def async_agent_call(agent: ChatAgent, messages: List[str]):
    """异步智能体调用"""
    tasks = []
    for msg in messages:
        task = asyncio.create_task(agent.astep(msg))
        tasks.append(task)

    responses = await asyncio.gather(*tasks)
    return responses

# 2. 使用流式响应
def stream_agent_response(agent: ChatAgent, message: str):
    """流式响应处理"""
    response = agent.step(message, stream=True)

    if isinstance(response, StreamingChatAgentResponse):
        for chunk in response:
            print(chunk.msgs[0].content, end="", flush=True)
    else:
        print(response.msgs[0].content)
```

## 8. 总结

CAMEL框架提供了丰富而灵活的API接口，支持：

1. **多种智能体类型**：从基础的ChatAgent到专业的任务智能体
2. **灵活的模型管理**：支持多平台模型和智能调度
3. **丰富的工具生态**：搜索、代码、图像处理等多种工具
4. **完善的记忆系统**：聊天历史、向量数据库等多种记忆方式
5. **强大的协作模式**：角色扮演、工作力系统等多种协作形式

通过合理使用这些API，开发者可以构建复杂的多智能体协作系统，实现各种AI应用场景。框架的设计遵循了良好的软件工程原则，确保了代码的可维护性、可扩展性和生产可用性。