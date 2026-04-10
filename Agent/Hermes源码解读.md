## tool call层：
### tool注册：
主要还是整体直接注册，实现在：自动扫描 (`_discover_tools`)和自注册 (`registry.register`)
自动扫描：
``` python
# model_tools.py (节选)


def _discover_tools():
    """
    触发工具发现：通过导入 tools/ 下的所有模块。
    每个工具文件内部会自动调用 registry.register()
    """
    _modules = [
        "tools.web_tools",
        "tools.terminal_tool",
        "tools.file_tools",
        # ... 其他工具
        "tools.delegate_tool",
    ]
    for mod_name in _modules:
        # 这个动作会让 Python 跑一遍该文件里的代码，从而触发下方的 register
        importlib.import_module(mod_name)

_discover_tools()
```
自注册：
``` python
# 假设这是 tools/example_tool.py
from tools.registry import registry
import json

# 真正的干活函数
def example_tool(param: str, task_id: str = None) -> str:
    return json.dumps({"success": True, "data": "任务完成"})

# 向档案库交简历：告诉系统我的名字、我的说明书（schema）、我要执行哪个函数（handler）
registry.register(
    name="example_tool",
    toolset="example", 
    schema={
        "name": "example_tool", 
        "description": "这是工具的说明书，发给大模型看的...",
        "parameters": {...}
    },
    handler=lambda args, **kw: example_tool(param=args.get("param", ""), task_id=kw.get("task_id")),
    # 可选：检查环境是否满足（比如有没有API Key）
    check_fn=check_requirements 
)

```

#### tool执行调用
llm收到拼装好的prompt，生成要调用的tool或者function call，会发送json格式命令：{"name": "read_file", "arguments": {"path": "test.txt"}}
首先会发送到
分层调度 (`run_agent.py`)这里会先看看发送的命令能否并行执行

中转执行 (`model_tools.handle_function_call`)
```python
# model_tools.py (节选，公开API)
def handle_function_call(function_name: str, function_args: Dict[str, Any],
                         task_id: Optional[str] = None,
                         user_task: Optional[str] = None) -> str:
    """
    负责把 function_name + arguments 变成真正的 handler 调用。
    """
    # 1. 参数清洗（防止模型乱传参数，比如把数字传成字符串）
    function_args = coerce_tool_args(function_name, function_args)
    
    # 2. 从注册表(registry)中找到对应的工具档案
    tool_record = registry.get_tool(function_name)
    
    if not tool_record:
        return f"Error: Tool {function_name} not found."

    # 3. 真正执行工具的 handler 函数，并获取结果
    try:
        # 这里就是去调用上面注册的 example_tool 等函数
        result = tool_record.handler(function_args, task_id=task_id) 
        return result # 必须返回字符串，因为要喂回给模型
    except Exception as e:
        return f"Tool Execution Error: {str(e)}"
```

