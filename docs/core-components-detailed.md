# CAMEL框架核心组件详解

## 目录

1. [Agents模块深度解析](#1-agents模块深度解析)
2. [Messages系统架构分析](#2-messages系统架构分析)
3. [Societies协作模块详解](#3-societies协作模块详解)
4. [Toolkits工具系统分析](#4-toolkits工具系统分析)
5. [Memories记忆系统架构](#5-memories记忆系统架构)
6. [Models模型抽象层](#6-models模型抽象层)
7. [Tasks任务管理系统](#7-tasks任务管理系统)

---

## 1. Agents模块深度解析

### 1.1 基础架构

#### BaseAgent抽象基类

```python
class BaseAgent(ABC):
    @abstractmethod
    def reset(self, *args: Any, **kwargs: Any) -> Any:
        """重置智能体到初始状态"""
        pass

    @abstractmethod
    def step(self, *args: Any, **kwargs: Any) -> Any:
        """执行单步操作"""
        pass
```

**设计要点：**
- 使用抽象基类确保所有智能体实现统一接口
- `reset()`方法用于状态重置和清理
- `step()`方法定义智能体的基本执行单元

#### 智能体类型体系

```
BaseAgent (抽象基类)
├── ChatAgent (核心聊天智能体)
│   ├── SearchAgent (搜索智能体)
│   ├── CriticAgent (评价智能体)
│   ├── DeductiveReasonerAgent (推理智能体)
│   └── TaskAgent系列 (任务智能体)
├── EmbodiedAgent (具身智能体)
├── KnowledgeGraphAgent (知识图谱智能体)
└── MCPAgent (MCP协议智能体)
```

### 1.2 ChatAgent核心实现

#### 核心特性

**状态管理**
- 支持记忆系统集成
- 工具调用状态跟踪
- 流式响应处理
- 会话上下文维护

**多模型支持**
- 通过ModelManager支持模型切换
- 负载均衡和容错切换
- 模型性能监控
- 动态模型选择

**工具集成**
- FunctionTool基础工具
- 外部工具包集成
- MCP工具协议支持
- 工具执行超时控制

**错误处理**
- 重试机制实现
- 超时控制
- 终止器模式
- 异常恢复策略

#### step()方法执行流程

```python
def step(self, input_message, response_format=None):
    # 1. 检查流式响应
    if stream:
        return StreamingChatAgentResponse(self._stream(input_message, response_format))

    # 2. 超时控制
    if step_timeout:
        with ThreadPoolExecutor() as executor:
            future = executor.submit(self._step_impl, input_message, response_format)
            return future.result(timeout=step_timeout)

    # 3. 执行核心逻辑
    return self._step_impl(input_message, response_format)
```

#### _step_impl核心逻辑

1. **输入处理阶段**
   - 消息格式转换和标准化
   - 添加到记忆系统
   - 上下文准备

2. **循环执行阶段**
   - 从记忆系统获取相关上下文
   - 调用模型获取响应
   - 处理工具调用请求
   - 检查终止条件
   - 更新智能体状态

3. **结果处理阶段**
   - 响应格式化
   - 工具调用记录
   - 记忆系统更新
   - 状态清理

### 1.3 工具调用机制

#### 工具系统架构

```python
class ToolSystem:
    def __init__(self):
        self._internal_tools = {}  # 内部工具注册表
        self._external_tools = []   # 外部工具包
        self.tool_calling_records = []  # 工具调用记录
```

#### 工具执行流程

```python
def _execute_tool(self, tool_call_request) -> ToolCallingRecord:
    # 1. 获取工具信息
    func_name = tool_call_request.tool_name
    args = tool_call_request.args
    tool = self._internal_tools[func_name]

    # 2. 执行工具
    try:
        raw_result = tool(**args)
        # 3. 处理输出掩码
        if self.mask_tool_output:
            result = "[工具执行成功，输出已屏蔽]"
        else:
            result = raw_result
    except Exception as e:
        result = f"工具执行失败: {e}"

    # 4. 记录工具调用
    return self._record_tool_calling(func_name, args, result, tool_call_id)
```

#### 异步工具执行

```python
async def _execute_tool_async(self, tool_call_request):
    # 1. 检查工具是否支持异步
    if hasattr(tool, 'async_call'):
        result = await tool.async_call(**args)
    elif asyncio.iscoroutinefunction(tool):
        result = await tool(**args)
    else:
        # 同步工具在线程池中执行
        loop = asyncio.get_event_loop()
        result = await loop.run_in_executor(None, tool, **args)

    return result
```

---

## 2. Messages系统架构分析

### 2.1 BaseMessage消息基类

#### 核心特性

**多模态支持**
```python
class BaseMessage:
    def __init__(
        self,
        content: Union[str, List[Union[str, Dict]]],
        role: str,
        role_name: str,
        meta: Optional[Dict] = None,
        **kwargs
    ):
        # 支持文本、图像、视频等多种内容类型
        self.content = content
        self.role = role
        self.role_name = role_name
        self.meta = meta or {}
```

**角色系统**
- `user`: 用户角色
- `assistant`: 助手角色
- `system`: 系统角色
- `tool`: 工具角色

**元数据管理**
```python
@property
def tool_call_ids(self) -> List[str]:
    """获取工具调用ID列表"""
    return self.meta.get("tool_call_ids", [])

@property
def tool_calls(self) -> List[Dict]:
    """获取工具调用信息"""
    return self.meta.get("tool_calls", [])
```

#### 消息类型体系

```
BaseMessage
├── FunctionCallingMessage (函数调用消息)
└── OpenAI格式消息系列
    ├── SystemMessage
    ├── UserMessage
    ├── AssistantMessage
    └── ToolMessage
```

### 2.2 消息转换机制

#### 设计模式应用

**策略模式**
```python
class MessageConverter:
    def __init__(self, target_format: str):
        self.target_format = target_format
        self.conversion_strategy = self._get_conversion_strategy()

    def convert(self, message: BaseMessage) -> Dict:
        return self.conversion_strategy.convert(message)
```

**适配器模式**
```python
class OpenAIAdapter:
    @staticmethod
    def to_openai_format(message: BaseMessage) -> Dict:
        """转换为OpenAI格式"""
        return {
            "role": message.role,
            "content": message.content,
            "name": message.role_name
        }
```

#### 核心转换功能

**OpenAI格式转换**
```python
def to_openai_messages(self, messages: List[BaseMessage]) -> List[Dict]:
    """批量转换为OpenAI格式"""
    openai_messages = []
    for msg in messages:
        if isinstance(msg, SystemMessage):
            openai_messages.append({"role": "system", "content": msg.content})
        elif isinstance(msg, UserMessage):
            openai_messages.append({"role": "user", "content": msg.content})
        # ... 其他类型处理
    return openai_messages
```

**ShareGPT格式转换**
```python
def to_sharegpt_format(self, conversation: List[BaseMessage]) -> Dict:
    """转换为ShareGPT格式"""
    return {
        "id": str(uuid.uuid4()),
        "conversations": [
            {
                "from": msg.role,
                "value": msg.content
            }
            for msg in conversation
        ]
    }
```

### 2.3 多模态消息处理

#### 图像消息处理

```python
class ImageMessage(BaseMessage):
    def __init__(self, image_path: str, **kwargs):
        self.image_path = image_path
        self.image_data = self._load_image(image_path)
        content = self._format_image_content()
        super().__init__(content=content, **kwargs)

    def _load_image(self, image_path: str) -> bytes:
        """加载图像数据"""
        with open(image_path, 'rb') as f:
            return f.read()

    def _format_image_content(self) -> List[Dict]:
        """格式化图像内容为OpenAI格式"""
        import base64
        encoded = base64.b64encode(self.image_data).decode('utf-8')
        return [
            {
                "type": "image_url",
                "image_url": {
                    "url": f"data:image/jpeg;base64,{encoded}"
                }
            }
        ]
```

#### 视频消息处理

```python
class VideoMessage(BaseMessage):
    def __init__(self, video_path: str, **kwargs):
        self.video_path = video_path
        self.video_frames = self._extract_frames(video_path)
        content = self._format_video_content()
        super().__init__(content=content, **kwargs)

    def _extract_frames(self, video_path: str) -> List[bytes]:
        """提取视频帧"""
        # 实现视频帧提取逻辑
        pass
```

---

## 3. Societies协作模块详解

### 3.1 RolePlaying角色扮演

#### 架构特点

**双智能体对话**
- 用户角色vs助手角色的结构化对话
- 角色定义和系统消息自动生成
- 对话历史管理和上下文维护

**评论家循环**
- 可选的CriticAgent参与质量评估
- 多轮反馈和改进机制
- 质量评分和建议生成

**任务规划**
- 可选的任务分解和规划
- 任务指定器（Task Specifier）
- 任务优先级管理

#### 实现模式

```python
class RolePlaying:
    def __init__(
        self,
        assistant_agent: ChatAgent,
        user_agent: ChatAgent,
        critic_agent: Optional[ChatAgent] = None,
        task_prompt: Optional[str] = None,
        with_task_specify: bool = False,
        with_task_planner: bool = False
    ):
        self.assistant_agent = assistant_agent
        self.user_agent = user_agent
        self.critic_agent = critic_agent
        self.task_prompt = task_prompt
        self.task_specify_agent = None
        self.task_planner_agent = None

        if with_task_specify:
            self.task_specify_agent = TaskSpecifyAgent()
        if with_task_planner:
            self.task_planner_agent = TaskPlannerAgent()

    def run(self) -> RolePlayingRecord:
        """执行角色扮演对话"""
        # 1. 任务规格化（可选）
        if self.task_specify_agent:
            specified_task = self._specify_task()
        else:
            specified_task = self.task_prompt

        # 2. 任务规划（可选）
        if self.task_planner_agent:
            planned_tasks = self._plan_tasks(specified_task)
        else:
            planned_tasks = [specified_task]

        # 3. 执行对话
        return self._execute_conversation(planned_tasks)
```

#### 对话执行流程

```python
def _execute_conversation(self, tasks: List[str]) -> RolePlayingRecord:
    conversation_records = []

    for task in tasks:
        # 初始化任务消息
        user_msg = BaseMessage.make_user_message(
            role_name="User",
            content=task
        )

        # 多轮对话循环
        while not self._should_terminate():
            # 助手响应
            assistant_response = self.assistant_agent.step(user_msg)
            conversation_records.append(assistant_response)

            # 用户响应
            user_response = self.user_agent.step(assistant_response.msg)
            conversation_records.append(user_response)

            # 评论家评估（可选）
            if self.critic_agent:
                critique = self.critic_agent.step(assistant_response.msg)
                if critique.should_improve:
                    # 改进对话
                    continue

            user_msg = user_response.msg

    return RolePlayingRecord(
        task_prompt=self.task_prompt,
        specified_task=specified_task,
        conversation_records=conversation_records
    )
```

### 3.2 Workforce工作力系统

#### 架构特点

**任务分发**
- 智能任务分配算法
- Worker能力匹配
- 任务优先级调度
- 负载均衡机制

**并行处理**
- 多Worker并行工作
- 任务依赖管理
- 结果聚合和合并
- 冲突解决机制

**错误恢复**
- 任务失败分析
- 自动重试机制
- Worker健康检查
- 降级处理策略

**资源管理**
- Worker池管理
- 资源使用监控
- 动态扩缩容
- 成本优化

#### Worker类型体系

```
BaseWorker (抽象基类)
├── SingleAgentWorker (单人worker)
├── RolePlayingWorker (双人worker)
└── CustomWorker (自定义worker)
```

#### 核心实现

```python
class Workforce:
    def __init__(
        self,
        workers: List[BaseWorker],
        storage: Optional[Storage] = None,
        task_prompt: Optional[str] = None,
        task_retries: int = 3,
        dependency_rule: str = "sequential"
    ):
        self.workers = workers
        self.storage = storage
        self.task_prompt = task_prompt
        self.task_retries = task_retries
        self.dependency_rule = dependency_rule

        # 任务管理器
        self.task_manager = TaskManager(
            retry_attempts=task_retries,
            dependency_rule=dependency_rule
        )

    def run(self) -> WorkforceRecord:
        """执行工作力任务"""
        # 1. 创建主任务
        main_task = self._create_main_task()

        # 2. 分解子任务
        subtasks = self._decompose_task(main_task)

        # 3. 分配任务给Workers
        task_assignments = self._assign_tasks(subtasks)

        # 4. 并行执行任务
        results = self._execute_tasks_parallel(task_assignments)

        # 5. 聚合结果
        aggregated_result = self._aggregate_results(results)

        return WorkforceRecord(
            task_prompt=self.task_prompt,
            task_records=results,
            final_result=aggregated_result
        )
```

### 3.3 BabyAGI自主系统

#### 核心特性

**自驱动任务创建**
- 基于当前状态自动生成新任务
- 任务优先级动态调整
- 目标导向的任务规划

**开放式研究循环**
- 研究和开发自动化
- 知识积累和应用
- 迭代改进机制

**最小化人类干预**
- 自主决策和执行
- 异常处理和恢复
- 自我优化和学习

#### 实现架构

```python
class BabyAGI:
    def __init__(
        self,
        objective: str,
        creation_agent: TaskCreationAgent,
        prioritization_agent: TaskPrioritizationAgent,
        execution_agent: ChatAgent,
        memory: Optional[AgentMemory] = None
    ):
        self.objective = objective
        self.creation_agent = creation_agent
        self.prioritization_agent = prioritization_agent
        self.execution_agent = execution_agent
        self.memory = memory or LongtermAgentMemory()

        # 任务队列
        self.task_queue = PriorityQueue()
        self.completed_tasks = []

        # 执行状态
        self.is_running = False
        self.max_iterations = 100

    def run(self) -> BabyAGIRecord:
        """执行BabyAGI循环"""
        self.is_running = True
        iteration = 0

        while self.is_running and iteration < self.max_iterations:
            # 1. 获取下一个任务
            if self.task_queue.empty():
                # 生成新任务
                new_tasks = self._generate_new_tasks()
                self._add_tasks_to_queue(new_tasks)

            current_task = self.task_queue.get()

            # 2. 执行任务
            task_result = self._execute_task(current_task)

            # 3. 更新记忆
            self._update_memory(current_task, task_result)

            # 4. 任务优先级重排序
            self._reprioritize_tasks()

            # 5. 检查终止条件
            if self._should_terminate():
                break

            iteration += 1

        return BabyAGIRecord(
            objective=self.objective,
            completed_tasks=self.completed_tasks,
            final_state=self.memory.get_current_state()
        )
```

---

## 4. Toolkits工具系统分析

### 4.1 BaseToolkit工具包基类

#### 设计特点

**MCP集成**
- 原生支持Model Context Protocol
- 标准化的工具接口
- 自动工具发现和注册

**超时管理**
```python
class BaseToolkit:
    def __init__(self, timeout: float = 30.0):
        self.timeout = timeout
        self._tools = {}
        self._add_timeout_decorators()

    def _add_timeout_decorators(self):
        """为所有方法添加超时装饰器"""
        for name, method in self.__class__.__dict__.items():
            if callable(method) and not name.startswith('_'):
                setattr(self, name, self._timeout_decorator(method))

    def _timeout_decorator(self, func):
        """超时装饰器"""
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            return timeout_decorator(self.timeout)(func)(*args, **kwargs)
        return wrapper
```

**工具注册**
```python
def add_tools(self, tools: List[FunctionTool]):
    """注册工具到工具包"""
    for tool in tools:
        self._tools[tool.name] = tool
        # 添加为实例方法
        setattr(self, tool.name, tool.function)
```

**Agent感知**
```python
class RegisteredAgentToolkit(BaseToolkit):
    def __init__(self, agent: ChatAgent):
        super().__init__()
        self.agent = agent
        self._register_agent_tools()

    def _register_agent_tools(self):
        """注册智能体相关工具"""
        self.add_tools([
            FunctionTool(self.agent_reset, "reset_agent"),
            FunctionTool(self.agent_memory, "get_agent_memory"),
            FunctionTool(self.agent_status, "get_agent_status")
        ])

    def agent_reset(self) -> str:
        """重置智能体状态"""
        self.agent.reset()
        return "Agent reset successfully"

    def agent_memory(self) -> Dict:
        """获取智能体记忆"""
        return self.agent.memory.get_records()

    def agent_status(self) -> Dict:
        """获取智能体状态"""
        return {
            "role_name": self.agent.role_name,
            "system_message": self.agent.system_message,
            "tools_count": len(self.agent.tools)
        }
```

### 4.2 工具生态系统

#### 核心工具包

**CodeExecutionToolkit**
```python
class CodeExecutionToolkit(BaseToolkit):
    def __init__(self):
        super().__init__()
        self.add_tools([
            FunctionTool(self.execute_python, "execute_python"),
            FunctionTool(self.execute_bash, "execute_bash"),
            FunctionTool(self.execute_sql, "execute_sql")
        ])

    def execute_python(self, code: str) -> str:
        """执行Python代码"""
        try:
            # 创建安全的执行环境
            exec_globals = {
                "__builtins__": {
                    "print": print,
                    "len": len,
                    "str": str,
                    "int": int,
                    "float": float,
                    "list": list,
                    "dict": dict,
                    "tuple": tuple
                }
            }

            # 执行代码
            exec_locals = {}
            exec(code, exec_globals, exec_locals)

            # 捕获输出
            output = io.StringIO()
            with redirect_stdout(output):
                exec(code, exec_globals, exec_locals)

            result = output.getvalue()
            return result if result else "Code executed successfully"

        except Exception as e:
            return f"Error executing Python code: {str(e)}"
```

**BrowserToolkit**
```python
class BrowserToolkit(BaseToolkit):
    def __init__(self):
        super().__init__()
        self.driver = None
        self.add_tools([
            FunctionTool(self.navigate_to, "navigate_to"),
            FunctionTool(self.click_element, "click_element"),
            FunctionTool(self.fill_form, "fill_form"),
            FunctionTool(self.extract_text, "extract_text"),
            FunctionTool(self.take_screenshot, "take_screenshot")
        ])

    def navigate_to(self, url: str) -> str:
        """导航到指定URL"""
        try:
            if not self.driver:
                self.driver = webdriver.Chrome()
            self.driver.get(url)
            return f"Navigated to {url}"
        except Exception as e:
            return f"Navigation failed: {str(e)}"

    def extract_text(self, selector: str = None) -> str:
        """提取页面文本"""
        try:
            if selector:
                element = self.driver.find_element(By.CSS_SELECTOR, selector)
                return element.text
            else:
                return self.driver.find_element(By.TAG_NAME, "body").text
        except Exception as e:
            return f"Text extraction failed: {str(e)}"
```

**SearchToolkit**
```python
class SearchToolkit(BaseToolkit):
    def __init__(self):
        super().__init__()
        self.add_tools([
            FunctionTool(self.search_duckduckgo, "search_duckduckgo"),
            FunctionTool(self.search_wikipedia, "search_wikipedia"),
            FunctionTool(self.search_arxiv, "search_arxiv")
        ])

    def search_duckduckgo(self, query: str, max_results: int = 5) -> str:
        """使用DuckDuckGo搜索"""
        try:
            from duckduckgo_search import DDGS

            results = []
            with DDGS() as ddgs:
                for r in ddgs.text(query, max_results=max_results):
                    results.append(f"{r['title']}\n{r['href']}\n{r['body']}\n")

            return "\n".join(results) if results else "No results found"

        except Exception as e:
            return f"Search failed: {str(e)}"

    def search_wikipedia(self, query: str, max_results: int = 3) -> str:
        """搜索维基百科"""
        try:
            import wikipedia

            # 搜索页面
            search_results = wikipedia.search(query, results=max_results)

            results = []
            for title in search_results:
                try:
                    page = wikipedia.page(title, auto_suggest=False)
                    results.append(f"Title: {page.title}\nURL: {page.url}\nSummary: {page.summary[:500]}...")
                except:
                    continue

            return "\n\n".join(results) if results else "No Wikipedia results found"

        except Exception as e:
            return f"Wikipedia search failed: {str(e)}"
```

#### 专业领域工具包

**GoogleCalendarToolkit**
```python
class GoogleCalendarToolkit(BaseToolkit):
    def __init__(self, credentials_path: str):
        super().__init__()
        self.credentials_path = credentials_path
        self.service = self._authenticate()
        self.add_tools([
            FunctionTool(self.create_event, "create_event"),
            FunctionTool(self.list_events, "list_events"),
            FunctionTool(self.update_event, "update_event"),
            FunctionTool(self.delete_event, "delete_event")
        ])

    def _authenticate(self):
        """Google API认证"""
        from google.oauth2.credentials import Credentials
        from googleapiclient.discovery import build

        creds = Credentials.from_authorized_user_file(self.credentials_path)
        service = build('calendar', 'v3', credentials=creds)
        return service

    def create_event(
        self,
        summary: str,
        start_time: str,
        end_time: str,
        description: str = "",
        attendees: List[str] = None
    ) -> str:
        """创建日历事件"""
        try:
            event = {
                'summary': summary,
                'description': description,
                'start': {
                    'dateTime': start_time,
                    'timeZone': 'UTC'
                },
                'end': {
                    'dateTime': end_time,
                    'timeZone': 'UTC'
                }
            }

            if attendees:
                event['attendees'] = [{'email': email} for email in attendees]

            event = self.service.events().insert(
                calendarId='primary',
                body=event
            ).execute()

            return f"Event created: {event.get('htmlLink')}"

        except Exception as e:
            return f"Failed to create event: {str(e)}"
```

**NetworkXToolkit**
```python
class NetworkXToolkit(BaseToolkit):
    def __init__(self):
        super().__init__()
        self.graphs = {}
        self.add_tools([
            FunctionTool(self.create_graph, "create_graph"),
            FunctionTool(self.add_node, "add_node"),
            FunctionTool(self.add_edge, "add_edge"),
            FunctionTool(self.analyze_graph, "analyze_graph"),
            FunctionTool(self.find_shortest_path, "find_shortest_path"),
            FunctionTool(self.calculate_centrality, "calculate_centrality")
        ])

    def create_graph(self, graph_id: str, graph_type: str = "undirected") -> str:
        """创建图"""
        try:
            if graph_type == "directed":
                self.graphs[graph_id] = nx.DiGraph()
            else:
                self.graphs[graph_id] = nx.Graph()

            return f"Created {graph_type} graph with ID: {graph_id}"

        except Exception as e:
            return f"Failed to create graph: {str(e)}"

    def analyze_graph(self, graph_id: str) -> str:
        """分析图的属性"""
        try:
            graph = self.graphs[graph_id]

            analysis = {
                "nodes": graph.number_of_nodes(),
                "edges": graph.number_of_edges(),
                "density": nx.density(graph),
                "is_connected": nx.is_connected(graph) if not graph.is_directed() else nx.is_strongly_connected(graph)
            }

            # 计算连通分量
            if not graph.is_directed():
                analysis["connected_components"] = nx.number_connected_components(graph)
            else:
                analysis["strongly_connected_components"] = nx.number_strongly_connected_components(graph)

            return f"Graph analysis:\n" + "\n".join([f"{k}: {v}" for k, v in analysis.items()])

        except Exception as e:
            return f"Graph analysis failed: {str(e)}"
```

### 4.3 MCP协议集成

#### MCP工具实现

```python
class MCPToolkit(BaseToolkit):
    def __init__(self, server_url: str):
        super().__init__()
        self.server_url = server_url
        self.client = MCPClient(server_url)
        self._discover_tools()

    def _discover_tools(self):
        """发现MCP服务器上的工具"""
        try:
            tools = self.client.list_tools()
            for tool in tools:
                # 创建MCP工具包装器
                mcp_tool = MCPTool(tool, self.client)
                self.add_tools([mcp_tool])
        except Exception as e:
            print(f"Failed to discover MCP tools: {e}")

    async def call_tool_async(self, tool_name: str, **kwargs) -> Any:
        """异步调用MCP工具"""
        try:
            return await self.client.call_tool(tool_name, **kwargs)
        except Exception as e:
            raise MCPToolError(f"MCP tool call failed: {e}")

class MCPTool(FunctionTool):
    def __init__(self, tool_def: Dict, client: MCPClient):
        self.tool_def = tool_def
        self.client = client
        super().__init__(self._execute_tool, tool_def["name"])

    async def _execute_tool(self, **kwargs) -> Any:
        """执行MCP工具"""
        return await self.client.call_tool(self.tool_def["name"], **kwargs)
```

---

## 5. Memories记忆系统架构

### 5.1 分层记忆架构

#### 记忆类型体系

```
AgentMemory (抽象记忆接口)
├── ChatHistoryMemory (聊天历史记忆)
│   ├── 窗口管理
│   └── 工具调用清理
├── VectorDBMemory (向量数据库记忆)
│   ├── 语义搜索
│   └── 相似性匹配
└── LongtermAgentMemory (长期记忆)
    ├── 信息压缩
    └── 重要性评分
```

#### AgentMemory抽象基类

```python
class AgentMemory(ABC):
    @abstractmethod
    def get_records(self, num: Optional[int] = None) -> List[MemoryRecord]:
        """获取记忆记录"""
        pass

    @abstractmethod
    def add_record(self, record: MemoryRecord) -> None:
        """添加记忆记录"""
        pass

    @abstractmethod
    def clear(self) -> None:
        """清空记忆"""
        pass

    @abstractmethod
    def get_context_messages(
        self,
        num: Optional[int] = None,
        token_limit: Optional[int] = None
    ) -> Tuple[List[BaseMessage], int]:
        """获取上下文消息"""
        pass
```

#### ChatHistoryMemory实现

```python
class ChatHistoryMemory(AgentMemory):
    def __init__(
        self,
        window_size: int = 1000,
        token_limit: int = 4000,
        enable_tool_calling_messages: bool = False
    ):
        self.window_size = window_size
        self.token_limit = token_limit
        self.enable_tool_calling_messages = enable_tool_calling_messages
        self._records: Deque[MemoryRecord] = deque(maxlen=window_size)

    def get_records(self, num: Optional[int] = None) -> List[MemoryRecord]:
        """获取记忆记录"""
        records = list(self._records)
        if num is not None:
            return records[-num:] if num > 0 else records[:num]
        return records

    def add_record(self, record: MemoryRecord) -> None:
        """添加记忆记录"""
        # 过滤工具调用消息
        if not self.enable_tool_calling_messages:
            if record.msg.role == "tool":
                return

        self._records.append(record)

    def get_context_messages(
        self,
        num: Optional[int] = None,
        token_limit: Optional[int] = None
    ) -> Tuple[List[BaseMessage], int]:
        """获取上下文消息"""
        token_limit = token_limit or self.token_limit
        messages = []
        total_tokens = 0

        # 获取记录
        records = self.get_records(num)

        for record in reversed(records):
            msg = record.msg
            msg_tokens = self._count_tokens(msg)

            if total_tokens + msg_tokens > token_limit:
                break

            messages.insert(0, msg)
            total_tokens += msg_tokens

        return messages, total_tokens

    def _count_tokens(self, message: BaseMessage) -> int:
        """计算消息的token数量"""
        # 简化的token计算，实际应使用tiktoken
        return len(str(message.content)) // 4
```

#### VectorDBMemory实现

```python
class VectorDBMemory(AgentMemory):
    def __init__(
        self,
        embedding_model: str = "text-embedding-ada-002",
        collection_name: str = "agent_memory",
        similarity_threshold: float = 0.7,
        max_results: int = 10
    ):
        self.embedding_model = embedding_model
        self.collection_name = collection_name
        self.similarity_threshold = similarity_threshold
        self.max_results = max_results

        # 初始化向量数据库
        self.client = ChromaClient()
        self.collection = self.client.get_or_create_collection(collection_name)

        # 初始化嵌入模型
        self.embedder = OpenAIEmbeddings(model=embedding_model)

    def add_record(self, record: MemoryRecord) -> None:
        """添加记忆记录到向量数据库"""
        # 生成嵌入向量
        text_content = str(record.msg.content)
        embedding = self.embedder.embed_query(text_content)

        # 添加到向量数据库
        self.collection.add(
            embeddings=[embedding],
            documents=[text_content],
            metadatas=[{
                "role": record.msg.role,
                "role_name": record.msg.role_name,
                "timestamp": record.timestamp.isoformat(),
                "message_type": type(record.msg).__name__
            }],
            ids=[str(record.uuid)]
        )

    def semantic_search(
        self,
        query: str,
        max_results: Optional[int] = None
    ) -> List[MemoryRecord]:
        """语义搜索相关记忆"""
        max_results = max_results or self.max_results

        # 生成查询嵌入
        query_embedding = self.embedder.embed_query(query)

        # 搜索相似文档
        results = self.collection.query(
            query_embeddings=[query_embedding],
            n_results=max_results,
            where={"similarity": {"$gte": self.similarity_threshold}}
        )

        # 转换为MemoryRecord
        records = []
        for i in range(len(results['documents'][0])):
            record = MemoryRecord(
                msg=BaseMessage(
                    content=results['documents'][0][i],
                    role=results['metadatas'][0][i]['role'],
                    role_name=results['metadatas'][0][i]['role_name']
                ),
                uuid=uuid.UUID(results['ids'][0][i]),
                timestamp=datetime.fromisoformat(results['metadatas'][0][i]['timestamp'])
            )
            records.append(record)

        return records
```

#### LongtermAgentMemory实现

```python
class LongtermAgentMemory(AgentMemory):
    def __init__(
        self,
        short_term_memory: ChatHistoryMemory,
        vector_memory: VectorDBMemory,
        compression_threshold: int = 100,
        importance_model: str = "gpt-3.5-turbo"
    ):
        self.short_term_memory = short_term_memory
        self.vector_memory = vector_memory
        self.compression_threshold = compression_threshold
        self.importance_model = importance_model

        # 压缩和重要性评分模型
        self.compression_model = ChatAgent(
            model=ModelFactory.create(
                model_platform=ModelPlatformType.OPENAI,
                model_type=ModelType.GPT_4,
                model_config_dict={"temperature": 0.3}
            )
        )

    def add_record(self, record: MemoryRecord) -> None:
        """添加记忆记录"""
        # 添加到短期记忆
        self.short_term_memory.add_record(record)

        # 添加到向量记忆
        self.vector_memory.add_record(record)

        # 检查是否需要压缩
        if len(self.short_term_memory.get_records()) >= self.compression_threshold:
            self._compress_memories()

    def _compress_memories(self):
        """压缩记忆"""
        records = self.short_term_memory.get_records()

        # 按重要性分组
        important_records = []
        normal_records = []

        for record in records:
            importance = self._calculate_importance(record)
            if importance > 0.7:
                important_records.append(record)
            else:
                normal_records.append(record)

        # 压缩普通记录
        if normal_records:
            compressed_summary = self._generate_summary(normal_records)
            compressed_record = MemoryRecord(
                msg=BaseMessage.make_system_message(
                    role_name="Memory Compression",
                    content=f"Previous conversation summary: {compressed_summary}"
                ),
                uuid=uuid.uuid4(),
                timestamp=datetime.now()
            )

            # 重置短期记忆，只保留重要记录和压缩记录
            self.short_term_memory.clear()
            for record in important_records:
                self.short_term_memory.add_record(record)
            self.short_term_memory.add_record(compressed_record)

    def _calculate_importance(self, record: MemoryRecord) -> float:
        """计算记忆重要性"""
        prompt = f"""
        Rate the importance of this memory record for future conversations:
        Role: {record.msg.role_name}
        Content: {record.msg.content}

        Importance score (0.0 to 1.0):
        """

        response = self.compression_model.step(
            BaseMessage.make_user_message(
                role_name="Importance Rater",
                content=prompt
            )
        )

        try:
            return float(response.msg.content.strip())
        except:
            return 0.5
```

### 5.2 上下文创建策略

#### ScoreBasedContextCreator

```python
class ScoreBasedContextCreator(BaseContextCreator):
    def __init__(
        self,
        token_limit: int = 4000,
        score_function: Optional[Callable] = None,
        decay_factor: float = 0.9,
        recency_weight: float = 0.3,
        importance_weight: float = 0.4,
        relevance_weight: float = 0.3
    ):
        self.token_limit = token_limit
        self.score_function = score_function or self._default_score_function
        self.decay_factor = decay_factor
        self.recency_weight = recency_weight
        self.importance_weight = importance_weight
        self.relevance_weight = relevance_weight

    def create_context(
        self,
        memories: List[MemoryRecord],
        current_message: Optional[BaseMessage] = None
    ) -> Tuple[List[BaseMessage], int]:
        """创建上下文"""
        # 1. 计算每个记忆的分数
        scored_memories = []
        for memory in memories:
            score = self._calculate_memory_score(memory, current_message)
            scored_memories.append((memory, score))

        # 2. 按分数排序
        scored_memories.sort(key=lambda x: x[1], reverse=True)

        # 3. 选择记忆直到达到token限制
        selected_memories = []
        total_tokens = 0

        for memory, score in scored_memories:
            msg = memory.msg
            msg_tokens = self._count_tokens(msg)

            if total_tokens + msg_tokens <= self.token_limit:
                selected_memories.append(msg)
                total_tokens += msg_tokens
            else:
                break

        return selected_memories, total_tokens

    def _calculate_memory_score(
        self,
        memory: MemoryRecord,
        current_message: Optional[BaseMessage] = None
    ) -> float:
        """计算记忆分数"""
        # 1. 新近性分数
        recency_score = self._calculate_recency_score(memory)

        # 2. 重要性分数
        importance_score = getattr(memory, 'importance_score', 0.5)

        # 3. 相关性分数
        relevance_score = self._calculate_relevance_score(memory, current_message)

        # 4. 综合分数
        total_score = (
            self.recency_weight * recency_score +
            self.importance_weight * importance_score +
            self.relevance_weight * relevance_score
        )

        return total_score

    def _calculate_recency_score(self, memory: MemoryRecord) -> float:
        """计算新近性分数"""
        time_diff = datetime.now() - memory.timestamp
        hours_diff = time_diff.total_seconds() / 3600

        # 指数衰减
        return self.decay_factor ** hours_diff

    def _calculate_relevance_score(
        self,
        memory: MemoryRecord,
        current_message: Optional[BaseMessage] = None
    ) -> float:
        """计算相关性分数"""
        if not current_message:
            return 0.5

        # 简单的关键词匹配
        memory_text = str(memory.msg.content).lower()
        current_text = str(current_message.content).lower()

        memory_words = set(memory_text.split())
        current_words = set(current_text.split())

        intersection = memory_words.intersection(current_words)
        union = memory_words.union(current_words)

        if not union:
            return 0.0

        return len(intersection) / len(union)
```

#### AdaptiveContextCreator

```python
class AdaptiveContextCreator(BaseContextCreator):
    def __init__(
        self,
        base_token_limit: int = 4000,
        max_token_limit: int = 8000,
        compression_threshold: float = 0.8,
        complexity_threshold: float = 0.7
    ):
        self.base_token_limit = base_token_limit
        self.max_token_limit = max_token_limit
        self.compression_threshold = compression_threshold
        self.complexity_threshold = complexity_threshold

    def create_context(
        self,
        memories: List[MemoryRecord],
        current_message: Optional[BaseMessage] = None
    ) -> Tuple[List[BaseMessage], int]:
        """自适应创建上下文"""
        # 1. 评估任务复杂性
        complexity = self._assess_task_complexity(current_message)

        # 2. 动态调整token限制
        token_limit = self._calculate_token_limit(complexity)

        # 3. 选择上下文创建策略
        if complexity > self.complexity_threshold:
            # 复杂任务：使用压缩策略
            return self._create_compressed_context(memories, token_limit)
        else:
            # 简单任务：使用标准策略
            return self._create_standard_context(memories, token_limit)

    def _assess_task_complexity(self, message: Optional[BaseMessage]) -> float:
        """评估任务复杂性"""
        if not message:
            return 0.5

        content = str(message.content).lower()

        complexity_indicators = [
            "analyze", "compare", "evaluate", "synthesize",
            "complex", "multiple", "integrate", "comprehensive"
        ]

        indicator_count = sum(1 for indicator in complexity_indicators
                             if indicator in content)

        return min(indicator_count / len(complexity_indicators), 1.0)

    def _calculate_token_limit(self, complexity: float) -> int:
        """根据复杂性计算token限制"""
        return int(
            self.base_token_limit +
            (self.max_token_limit - self.base_token_limit) * complexity
        )

    def _create_compressed_context(
        self,
        memories: List[MemoryRecord],
        token_limit: int
    ) -> Tuple[List[BaseMessage], int]:
        """创建压缩上下文"""
        # 1. 分组记忆
        recent_memories = memories[-10:]  # 最近10条
        old_memories = memories[:-10]

        # 2. 压缩旧记忆
        if old_memories:
            summary = self._generate_memory_summary(old_memories)
            summary_message = BaseMessage.make_system_message(
                role_name="Memory Summary",
                content=f"Previous context summary: {summary}"
            )
            selected_memories = [summary_message] + recent_memories
        else:
            selected_memories = recent_memories

        # 3. 计算token
        total_tokens = sum(self._count_tokens(msg) for msg in selected_memories)

        return selected_memories, total_tokens
```

---

## 6. Models模型抽象层

### 6.1 BaseModelBackend模型后端

#### 核心特性

**统一接口**
```python
class BaseModelBackend(ABC, metaclass=BackendMeta):
    @abstractmethod
    def run(
        self,
        messages: List[OpenAIMessage],
        **kwargs
    ) -> ModelResponse:
        """同步调用模型"""
        pass

    @abstractmethod
    async def arun(
        self,
        messages: List[OpenAIMessage],
        **kwargs
    ) -> ModelResponse:
        """异步调用模型"""
        pass

    @abstractmethod
    def check_model_config(self, model_config: Dict[str, Any]) -> None:
        """检查模型配置"""
        pass

    @abstractmethod
    def get_token_limit(self) -> int:
        """获取模型token限制"""
        pass
```

**元类增强**
```python
class BackendMeta(type):
    def __new__(cls, name, bases, namespace):
        new_class = super().__new__(cls, name, bases, namespace)

        # 自动添加消息预处理
        original_run = getattr(new_class, 'run', None)
        if original_run:
            def wrapped_run(self, messages, **kwargs):
                processed_messages = self._preprocess_messages(messages)
                return original_run(self, processed_messages, **kwargs)
            setattr(new_class, 'run', wrapped_run)

        return new_class
```

**流式支持**
```python
def stream(
    self,
    messages: List[OpenAIMessage],
    **kwargs
) -> Iterator[ModelResponse]:
    """流式调用模型"""
    try:
        response = self.client.chat.completions.create(
            model=self.model_type,
            messages=messages,
            stream=True,
            **kwargs
        )

        for chunk in response:
            if chunk.choices[0].delta.content:
                yield ModelResponse(
                    content=chunk.choices[0].delta.content,
                    finish_reason=chunk.choices[0].finish_reason,
                    usage=None
                )

    except Exception as e:
        raise ModelBackendError(f"Streaming failed: {e}")
```

**错误处理**
```python
class BaseModelBackend(ABC, metaclass=BackendMeta):
    def __init__(
        self,
        model_type: str,
        model_config_dict: Dict[str, Any],
        retry_attempts: int = 3,
        retry_delay: float = 1.0
    ):
        self.model_type = model_type
        self.model_config_dict = model_config_dict
        self.retry_attempts = retry_attempts
        self.retry_delay = retry_delay

        # 初始化模型客户端
        self.client = self._initialize_client()

    def run_with_retry(
        self,
        messages: List[OpenAIMessage],
        **kwargs
    ) -> ModelResponse:
        """带重试的模型调用"""
        last_error = None

        for attempt in range(self.retry_attempts):
            try:
                return self.run(messages, **kwargs)
            except RateLimitError as e:
                last_error = e
                if attempt < self.retry_attempts - 1:
                    delay = self._calculate_retry_delay(attempt)
                    time.sleep(delay)
                continue
            except Exception as e:
                last_error = e
                if attempt < self.retry_attempts - 1:
                    time.sleep(self.retry_delay)
                continue

        raise ModelBackendError(f"Model call failed after {self.retry_attempts} attempts: {last_error}")

    def _calculate_retry_delay(self, attempt: int) -> float:
        """计算重试延迟（指数退避）"""
        return min(self.retry_delay * (2 ** attempt), 60.0)
```

### 6.2 OpenAIModelBackend实现

```python
class OpenAIModelBackend(BaseModelBackend):
    def __init__(self, model_type: str, model_config_dict: Dict[str, Any]):
        super().__init__(model_type, model_config_dict)
        self.api_key = model_config_dict.get("api_key")
        self.base_url = model_config_dict.get("base_url")
        self.organization = model_config_dict.get("organization")

        # 初始化OpenAI客户端
        self.client = openai.OpenAI(
            api_key=self.api_key,
            base_url=self.base_url,
            organization=self.organization
        )

    def run(
        self,
        messages: List[OpenAIMessage],
        temperature: Optional[float] = None,
        max_tokens: Optional[int] = None,
        top_p: Optional[float] = None,
        stop: Optional[Union[str, List[str]]] = None,
        **kwargs
    ) -> ModelResponse:
        """调用OpenAI模型"""
        try:
            # 构建请求参数
            request_params = {
                "model": self.model_type,
                "messages": messages,
                "temperature": temperature or self.model_config_dict.get("temperature", 0.7),
                "max_tokens": max_tokens or self.model_config_dict.get("max_tokens"),
                "top_p": top_p or self.model_config_dict.get("top_p", 1.0),
                **kwargs
            }

            if stop:
                request_params["stop"] = stop

            # 调用模型
            response = self.client.chat.completions.create(**request_params)

            # 转换响应
            return ModelResponse(
                content=response.choices[0].message.content,
                finish_reason=response.choices[0].finish_reason,
                usage=ModelUsage(
                    prompt_tokens=response.usage.prompt_tokens,
                    completion_tokens=response.usage.completion_tokens,
                    total_tokens=response.usage.total_tokens
                ),
                cost=self._calculate_cost(response.usage)
            )

        except Exception as e:
            raise ModelBackendError(f"OpenAI model call failed: {e}")

    async def arun(
        self,
        messages: List[OpenAIMessage],
        **kwargs
    ) -> ModelResponse:
        """异步调用OpenAI模型"""
        try:
            # 使用asyncio线程池执行同步调用
            loop = asyncio.get_event_loop()
            return await loop.run_in_executor(
                None,
                lambda: self.run(messages, **kwargs)
            )
        except Exception as e:
            raise ModelBackendError(f"OpenAI async model call failed: {e}")

    def get_token_limit(self) -> int:
        """获取模型token限制"""
        token_limits = {
            "gpt-3.5-turbo": 4096,
            "gpt-3.5-turbo-16k": 16384,
            "gpt-4": 8192,
            "gpt-4-32k": 32768,
            "gpt-4-turbo": 128000,
            "gpt-4o": 128000,
            "gpt-4o-mini": 128000
        }
        return token_limits.get(self.model_type, 4096)

    def _calculate_cost(self, usage: Any) -> float:
        """计算API调用成本"""
        # OpenAI定价模型（示例）
        pricing = {
            "gpt-3.5-turbo": {"input": 0.0015, "output": 0.002},
            "gpt-4": {"input": 0.03, "output": 0.06},
            "gpt-4-turbo": {"input": 0.01, "output": 0.03},
            "gpt-4o": {"input": 0.005, "output": 0.015},
            "gpt-4o-mini": {"input": 0.00015, "output": 0.0006}
        }

        model_pricing = pricing.get(self.model_type, {"input": 0.001, "output": 0.002})

        input_cost = (usage.prompt_tokens / 1000) * model_pricing["input"]
        output_cost = (usage.completion_tokens / 1000) * model_pricing["output"]

        return input_cost + output_cost
```

### 6.3 ModelFactory工厂模式

```python
class ModelFactory:
    _model_registry = {}

    @classmethod
    def register_model(
        cls,
        platform: ModelPlatformType,
        model_type: ModelType,
        model_backend_class: Type[BaseModelBackend]
    ):
        """注册模型后端"""
        if platform not in cls._model_registry:
            cls._model_registry[platform] = {}
        cls._model_registry[platform][model_type] = model_backend_class

    @classmethod
    def create(
        cls,
        model_platform: ModelPlatformType,
        model_type: ModelType,
        model_config_dict: Optional[Dict[str, Any]] = None,
        **kwargs
    ) -> BaseModelBackend:
        """创建模型后端实例"""
        if model_config_dict is None:
            model_config_dict = {}

        # 检查平台是否支持
        if model_platform not in cls._model_registry:
            raise ValueError(f"Unsupported model platform: {model_platform}")

        # 检查模型类型是否支持
        if model_type not in cls._model_registry[model_platform]:
            raise ValueError(f"Unsupported model type: {model_type} for platform: {model_platform}")

        # 获取模型后端类
        model_backend_class = cls._model_registry[model_platform][model_type]

        # 合并配置参数
        final_config = {**model_config_dict, **kwargs}

        # 创建模型后端实例
        try:
            return model_backend_class(
                model_type=model_type.value,
                model_config_dict=final_config
            )
        except Exception as e:
            raise ModelFactoryError(f"Failed to create model backend: {e}")

    @classmethod
    def get_supported_platforms(cls) -> List[ModelPlatformType]:
        """获取支持的模型平台"""
        return list(cls._model_registry.keys())

    @classmethod
    def get_supported_models(cls, platform: ModelPlatformType) -> List[ModelType]:
        """获取平台支持的模型类型"""
        if platform not in cls._model_registry:
            return []
        return list(cls._model_registry[platform].keys())

# 注册内置模型后端
ModelFactory.register_model(
    ModelPlatformType.OPENAI,
    ModelType.GPT_4O,
    OpenAIModelBackend
)

ModelFactory.register_model(
    ModelPlatformType.OPENAI,
    ModelType.GPT_4O_MINI,
    OpenAIModelBackend
)

ModelFactory.register_model(
    ModelPlatformType.ANTHROPIC,
    ModelType.CLAUDE_3_OPUS,
    AnthropicModelBackend
)
```

### 6.4 ModelManager管理器

```python
class ModelManager:
    def __init__(
        self,
        models: List[BaseModelBackend],
        scheduling_strategy: str = "round_robin",
        health_check_interval: float = 60.0
    ):
        self.models = models
        self.scheduling_strategy = scheduling_strategy
        self.health_check_interval = health_check_interval

        # 调度状态
        self.current_index = 0
        self.model_health = {model: True for model in models}
        self.model_stats = {model: ModelStats() for model in models}

        # 启动健康检查
        self._start_health_check()

    def get_model(self) -> BaseModelBackend:
        """获取下一个可用模型"""
        available_models = [
            model for model in self.models
            if self.model_health.get(model, False)
        ]

        if not available_models:
            raise ModelManagerError("No available models")

        if self.scheduling_strategy == "round_robin":
            return self._round_robin_schedule(available_models)
        elif self.scheduling_strategy == "random":
            return self._random_schedule(available_models)
        elif self.scheduling_strategy == "least_loaded":
            return self._least_loaded_schedule(available_models)
        elif self.scheduling_strategy == "always_first":
            return available_models[0]
        else:
            return available_models[0]

    def _round_robin_schedule(self, models: List[BaseModelBackend]) -> BaseModelBackend:
        """轮询调度"""
        model = models[self.current_index % len(models)]
        self.current_index += 1
        return model

    def _random_schedule(self, models: List[BaseModelBackend]) -> BaseModelBackend:
        """随机调度"""
        return random.choice(models)

    def _least_loaded_schedule(self, models: List[BaseModelBackend]) -> BaseModelBackend:
        """最小负载调度"""
        return min(models, key=lambda m: self.model_stats[m].current_load)

    def update_model_stats(self, model: BaseModelBackend, stats: ModelStats):
        """更新模型统计信息"""
        if model in self.model_stats:
            self.model_stats[model] = stats

    def _start_health_check(self):
        """启动健康检查"""
        def health_check_loop():
            while True:
                self._perform_health_check()
                time.sleep(self.health_check_interval)

        thread = threading.Thread(target=health_check_loop, daemon=True)
        thread.start()

    def _perform_health_check(self):
        """执行健康检查"""
        for model in self.models:
            try:
                # 简单的健康检查调用
                test_messages = [{"role": "user", "content": "test"}]
                response = model.run(test_messages, max_tokens=1)

                if response.content:
                    self.model_health[model] = True
                else:
                    self.model_health[model] = False

            except Exception:
                self.model_health[model] = False

class ModelStats:
    def __init__(self):
        self.total_calls = 0
        self.successful_calls = 0
        self.failed_calls = 0
        self.average_latency = 0.0
        self.current_load = 0.0
        self.last_used = None

    def update_call(self, success: bool, latency: float):
        """更新调用统计"""
        self.total_calls += 1
        if success:
            self.successful_calls += 1
        else:
            self.failed_calls += 1

        # 更新平均延迟
        if self.total_calls == 1:
            self.average_latency = latency
        else:
            self.average_latency = (
                (self.average_latency * (self.total_calls - 1) + latency) /
                self.total_calls
            )

        self.last_used = datetime.now()

    @property
    def success_rate(self) -> float:
        """成功率"""
        if self.total_calls == 0:
            return 0.0
        return self.successful_calls / self.total_calls
```

---

## 7. Tasks任务管理系统

### 7.1 Task任务实体

#### 特性

**状态管理**
```python
class TaskState(Enum):
    TODO = "todo"           # 待处理
    RUNNING = "running"     # 运行中
    DONE = "done"          # 已完成
    FAILED = "failed"      # 失败
    CANCELLED = "cancelled" # 已取消

class Task:
    def __init__(
        self,
        task_id: str,
        content: str,
        dependencies: Optional[List[str]] = None,
        validation_mode: TaskValidationMode = TaskValidationMode.AUTO,
        priority: int = 0,
        timeout: Optional[float] = None,
        retry_attempts: int = 3
    ):
        self.task_id = task_id
        self.content = content
        self.dependencies = dependencies or []
        self.validation_mode = validation_mode
        self.priority = priority
        self.timeout = timeout
        self.retry_attempts = retry_attempts

        # 状态属性
        self.state = TaskState.TODO
        self.result = None
        self.error = None
        self.start_time = None
        self.end_time = None
        self.attempts = 0

        # 子任务
        self.subtasks: List[Task] = []
        self.parent_task: Optional[Task] = None

    def add_subtask(self, subtask: 'Task'):
        """添加子任务"""
        subtask.parent_task = self
        self.subtasks.append(subtask)

    def mark_started(self):
        """标记任务开始"""
        self.state = TaskState.RUNNING
        self.start_time = datetime.now()
        self.attempts += 1

    def mark_completed(self, result: Any):
        """标记任务完成"""
        self.state = TaskState.DONE
        self.result = result
        self.end_time = datetime.now()

    def mark_failed(self, error: Exception):
        """标记任务失败"""
        self.state = TaskState.FAILED
        self.error = error
        self.end_time = datetime.now()

    def can_execute(self, completed_tasks: Set[str]) -> bool:
        """检查任务是否可以执行"""
        if self.state != TaskState.TODO:
            return False

        # 检查依赖是否都已完成
        return all(dep_id in completed_tasks for dep_id in self.dependencies)

    def get_duration(self) -> Optional[float]:
        """获取任务执行时长"""
        if self.start_time and self.end_time:
            return (self.end_time - self.start_time).total_seconds()
        return None

    def __repr__(self):
        return f"Task(id={self.task_id}, state={self.state.value}, priority={self.priority})"
```

**依赖关系**
```python
class TaskDependency:
    def __init__(
        self,
        task_id: str,
        depends_on: List[str],
        dependency_type: DependencyType = DependencyType.COMPLETION
    ):
        self.task_id = task_id
        self.depends_on = depends_on
        self.dependency_type = dependency_type

    def is_satisfied(self, task_states: Dict[str, TaskState]) -> bool:
        """检查依赖是否满足"""
        if self.dependency_type == DependencyType.COMPLETION:
            return all(
                task_states.get(dep_id) == TaskState.DONE
                for dep_id in self.depends_on
            )
        elif self.dependency_type == DependencyType.SUCCESS:
            return all(
                task_states.get(dep_id) == TaskState.DONE
                for dep_id in self.depends_on
            )
        elif self.dependency_type == DependencyType.FAILURE:
            return any(
                task_states.get(dep_id) == TaskState.FAILED
                for dep_id in self.depends_on
            )

        return False

class DependencyType(Enum):
    COMPLETION = "completion"  # 任务完成
    SUCCESS = "success"       # 任务成功
    FAILURE = "failure"       # 任务失败
    TIMEOUT = "timeout"       # 任务超时
```

**验证机制**
```python
class TaskValidationMode(Enum):
    AUTO = "auto"           # 自动验证
    MANUAL = "manual"       # 手动验证
    NONE = "none"           # 无验证

class TaskValidator:
    def __init__(self, validation_mode: TaskValidationMode):
        self.validation_mode = validation_mode

    def validate_task(self, task: Task, result: Any) -> ValidationResult:
        """验证任务结果"""
        if self.validation_mode == TaskValidationMode.NONE:
            return ValidationResult(True, "No validation required")

        elif self.validation_mode == TaskValidationMode.AUTO:
            return self._auto_validate(task, result)

        elif self.validation_mode == TaskValidationMode.MANUAL:
            return self._manual_validate(task, result)

        return ValidationResult(False, "Unknown validation mode")

    def _auto_validate(self, task: Task, result: Any) -> ValidationResult:
        """自动验证任务结果"""
        try:
            # 基于任务类型的验证逻辑
            if "code" in task.content.lower():
                return self._validate_code_result(task, result)
            elif "search" in task.content.lower():
                return self._validate_search_result(task, result)
            else:
                return self._validate_general_result(task, result)

        except Exception as e:
            return ValidationResult(False, f"Validation error: {e}")

    def _validate_code_result(self, task: Task, result: Any) -> ValidationResult:
        """验证代码执行结果"""
        if isinstance(result, str) and "error" in result.lower():
            return ValidationResult(False, f"Code execution failed: {result}")
        return ValidationResult(True, "Code validation passed")

    def _validate_search_result(self, task: Task, result: Any) -> ValidationResult:
        """验证搜索结果"""
        if isinstance(result, str) and len(result.strip()) == 0:
            return ValidationResult(False, "Empty search result")
        return ValidationResult(True, "Search validation passed")

    def _validate_general_result(self, task: Task, result: Any) -> ValidationResult:
        """验证一般任务结果"""
        if result is None:
            return ValidationResult(False, "Empty result")
        return ValidationResult(True, "General validation passed")

class ValidationResult:
    def __init__(self, is_valid: bool, message: str):
        self.is_valid = is_valid
        self.message = message

    def __bool__(self):
        return self.is_valid
```

### 7.2 TaskManager任务管理器

#### 功能

**任务生命周期管理**
```python
class TaskManager:
    def __init__(
        self,
        max_concurrent_tasks: int = 5,
        retry_attempts: int = 3,
        dependency_rule: str = "sequential"
    ):
        self.max_concurrent_tasks = max_concurrent_tasks
        self.retry_attempts = retry_attempts
        self.dependency_rule = dependency_rule

        # 任务存储
        self.tasks: Dict[str, Task] = {}
        self.task_queue = PriorityQueue()
        self.running_tasks: Dict[str, Task] = {}
        self.completed_tasks: Dict[str, Task] = {}
        self.failed_tasks: Dict[str, Task] = {}

        # 执行器
        self.executor = ThreadPoolExecutor(max_workers=max_concurrent_tasks)

        # 统计信息
        self.stats = TaskManagerStats()

        # 事件处理
        self.task_listeners: List[TaskListener] = []

    def add_task(self, task: Task):
        """添加任务"""
        self.tasks[task.task_id] = task
        self.task_queue.put((task.priority, task.task_id))
        self._notify_task_added(task)

    def add_tasks(self, tasks: List[Task]):
        """批量添加任务"""
        for task in tasks:
            self.add_task(task)

    def execute_tasks(self) -> Dict[str, Any]:
        """执行所有任务"""
        futures = {}

        while not self.task_queue.empty() or self.running_tasks:
            # 检查是否有可以开始的任务
            self._start_ready_tasks()

            # 检查任务完成情况
            self._check_task_completion(futures)

            # 短暂休眠避免CPU占用过高
            time.sleep(0.1)

        # 等待所有任务完成
        for future in futures.values():
            future.result()

        return self._generate_execution_summary()

    def _start_ready_tasks(self):
        """启动准备就绪的任务"""
        while len(self.running_tasks) < self.max_concurrent_tasks and not self.task_queue.empty():
            priority, task_id = self.task_queue.get()
            task = self.tasks[task_id]

            if task.can_execute(set(self.completed_tasks.keys())):
                # 启动任务
                future = self.executor.submit(self._execute_task, task)
                self.running_tasks[task_id] = task
                self._notify_task_started(task)

    def _execute_task(self, task: Task) -> Task:
        """执行单个任务"""
        try:
            task.mark_started()

            # 执行任务
            result = self._run_task_with_timeout(task)

            # 验证结果
            validator = TaskValidator(task.validation_mode)
            validation_result = validator.validate_task(task, result)

            if validation_result.is_valid:
                task.mark_completed(result)
                self.completed_tasks[task.task_id] = task
                self._notify_task_completed(task)
            else:
                task.mark_failed(Exception(validation_result.message))
                self.failed_tasks[task.task_id] = task
                self._notify_task_failed(task)

        except Exception as e:
            task.mark_failed(e)
            self.failed_tasks[task.task_id] = task
            self._notify_task_failed(task)

        finally:
            self.running_tasks.pop(task.task_id, None)

        return task

    def _run_task_with_timeout(self, task: Task) -> Any:
        """带超时的任务执行"""
        if task.timeout is None:
            return self._run_task_content(task)

        try:
            return self.executor.submit(self._run_task_content, task).result(timeout=task.timeout)
        except TimeoutError:
            raise TaskTimeoutError(f"Task {task.task_id} timed out")

    def _run_task_content(self, task: Task) -> Any:
        """执行任务内容"""
        # 这里应该根据任务类型执行相应的逻辑
        # 示例：简单的任务内容处理
        if "calculate" in task.content.lower():
            return self._execute_calculation_task(task)
        elif "search" in task.content.lower():
            return self._execute_search_task(task)
        else:
            return self._execute_general_task(task)

    def _execute_calculation_task(self, task: Task) -> Any:
        """执行计算任务"""
        # 简单的计算示例
        try:
            # 假设任务内容包含数学表达式
            expression = task.content.split("calculate")[-1].strip()
            result = eval(expression)  # 注意：实际应用中应使用更安全的表达式求值
            return result
        except Exception as e:
            raise TaskExecutionError(f"Calculation failed: {e}")

    def _execute_search_task(self, task: Task) -> Any:
        """执行搜索任务"""
        # 搜索任务示例
        try:
            query = task.content.split("search")[-1].strip()
            # 这里应该调用实际的搜索API
            return f"Search results for: {query}"
        except Exception as e:
            raise TaskExecutionError(f"Search failed: {e}")

    def _execute_general_task(self, task: Task) -> Any:
        """执行一般任务"""
        # 一般任务处理
        try:
            # 简单的任务处理逻辑
            return f"Task '{task.content}' completed successfully"
        except Exception as e:
            raise TaskExecutionError(f"General task failed: {e}")
```

**依赖解析**
```python
class DependencyResolver:
    def __init__(self, tasks: Dict[str, Task]):
        self.tasks = tasks
        self.graph = self._build_dependency_graph()

    def _build_dependency_graph(self) -> Dict[str, List[str]]:
        """构建依赖图"""
        graph = {}
        for task_id, task in self.tasks.items():
            graph[task_id] = task.dependencies
        return graph

    def resolve_dependencies(self) -> List[List[str]]:
        """解析任务依赖，返回可并行执行的任务组"""
        try:
            # 拓扑排序
            return self._topological_sort()
        except CycleError as e:
            raise TaskDependencyError(f"Circular dependency detected: {e}")

    def _topological_sort(self) -> List[List[str]]:
        """拓扑排序，返回可并行执行的任务组"""
        # 计算入度
        in_degree = {task_id: 0 for task_id in self.graph}
        for task_id in self.graph:
            for dep in self.graph[task_id]:
                if dep in in_degree:
                    in_degree[dep] += 1

        # 初始化队列
        queue = deque([task_id for task_id, degree in in_degree.items() if degree == 0])
        result = []

        while queue:
            # 当前层级的所有任务可以并行执行
            current_level = list(queue)
            result.append(current_level)
            queue.clear()

            # 更新入度
            for task_id in current_level:
                for dep in self.graph.get(task_id, []):
                    if dep in in_degree:
                        in_degree[dep] -= 1
                        if in_degree[dep] == 0:
                            queue.append(dep)

        # 检查是否有环
        if len([task_id for task_id, degree in in_degree.items() if degree > 0]) > 0:
            raise CycleError("Circular dependency detected")

        return result

    def get_critical_path(self) -> List[str]:
        """获取关键路径"""
        # 这里实现关键路径算法
        # 简化版本：返回最长依赖链
        longest_path = []
        for task_id in self.graph:
            path = self._get_longest_path(task_id)
            if len(path) > len(longest_path):
                longest_path = path

        return longest_path

    def _get_longest_path(self, task_id: str, visited: Optional[Set[str]] = None) -> List[str]:
        """获取从指定任务开始的最长路径"""
        if visited is None:
            visited = set()

        if task_id in visited:
            return []

        visited.add(task_id)
        longest_path = [task_id]

        for dep in self.graph.get(task_id, []):
            dep_path = self._get_longest_path(dep, visited.copy())
            if len(dep_path) > len(longest_path) - 1:
                longest_path = dep_path + [task_id]

        return longest_path

class CycleError(Exception):
    pass

class TaskDependencyError(Exception):
    pass
```

**状态跟踪**
```python
class TaskManagerStats:
    def __init__(self):
        self.total_tasks = 0
        self.completed_tasks = 0
        self.failed_tasks = 0
        self.running_tasks = 0
        self.pending_tasks = 0

        self.total_execution_time = 0.0
        self.average_execution_time = 0.0
        self.max_execution_time = 0.0
        self.min_execution_time = float('inf')

        self.task_start_times = {}
        self.task_end_times = {}

    def update_task_started(self, task: Task):
        """更新任务开始统计"""
        self.total_tasks += 1
        self.running_tasks += 1
        self.pending_tasks -= 1
        self.task_start_times[task.task_id] = datetime.now()

    def update_task_completed(self, task: Task):
        """更新任务完成统计"""
        self.completed_tasks += 1
        self.running_tasks -= 1

        end_time = datetime.now()
        self.task_end_times[task.task_id] = end_time

        if task.task_id in self.task_start_times:
            start_time = self.task_start_times[task.task_id]
            duration = (end_time - start_time).total_seconds()

            self.total_execution_time += duration
            self.average_execution_time = self.total_execution_time / self.completed_tasks
            self.max_execution_time = max(self.max_execution_time, duration)
            self.min_execution_time = min(self.min_execution_time, duration)

    def update_task_failed(self, task: Task):
        """更新任务失败统计"""
        self.failed_tasks += 1
        self.running_tasks -= 1

    def get_completion_rate(self) -> float:
        """获取完成率"""
        if self.total_tasks == 0:
            return 0.0
        return self.completed_tasks / self.total_tasks

    def get_failure_rate(self) -> float:
        """获取失败率"""
        if self.total_tasks == 0:
            return 0.0
        return self.failed_tasks / self.total_tasks

    def get_summary(self) -> Dict[str, Any]:
        """获取统计摘要"""
        return {
            "total_tasks": self.total_tasks,
            "completed_tasks": self.completed_tasks,
            "failed_tasks": self.failed_tasks,
            "running_tasks": self.running_tasks,
            "pending_tasks": self.pending_tasks,
            "completion_rate": self.get_completion_rate(),
            "failure_rate": self.get_failure_rate(),
            "average_execution_time": self.average_execution_time,
            "max_execution_time": self.max_execution_time,
            "min_execution_time": self.min_execution_time if self.min_execution_time != float('inf') else 0
        }

class TaskListener(ABC):
    @abstractmethod
    def on_task_added(self, task: Task):
        pass

    @abstractmethod
    def on_task_started(self, task: Task):
        pass

    @abstractmethod
    def on_task_completed(self, task: Task):
        pass

    @abstractmethod
    def on_task_failed(self, task: Task):
        pass
```

---

## 总结

本文档深入分析了CAMEL框架的七个核心模块，详细阐述了每个模块的架构设计、实现细节和关键特性。通过这些模块的有机组合，CAMEL框架构建了一个功能完整、扩展性强的多智能体系统基础设施。

每个模块都采用了良好的设计模式和架构原则，确保了代码的可维护性、可扩展性和生产可用性。这些核心模块为构建复杂的多智能体应用提供了强大的技术支撑。