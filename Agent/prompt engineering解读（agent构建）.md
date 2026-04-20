
### Point1  做好prompt enginneering的第一步
最开始的prompt理解，一个prompt-builder文件下，拼接user prompt + system prompt + 选点memory
这样虽然基本够用，但是就跟最简单的while agent loop一样，没有技术含量，没有engineering。
我过去以为prompt是专门构建给llm看的上下文，我们要做好prompt的拼接和内容管理。错误的把关注点放在了如何“写好、拼接”prompt上。但实际上，在做好prompt engineering的第一步不是在prompt-builder上，而是在runtime维护好一个**state**。
所以真的要往可持续运行、边界清晰的上下文 agent loop走，我们不能把prompt看成runtime内部依赖的一个描述文本，而是runtime state 投影成llm能看懂的上下文。

自检方法：
**如果我把 prompt 文案整体换一种写法，但不改 session/state，系统还能稳定运行吗？**
- 能：说明 prompt 更像投影
- 不能：说明 prompt 偷偷承担了状态职责

### Point2  prompt不能按内容主题划分，而是变化速度划分
一个好的prompt system，不仅要从llm能否拿到足够的上下文来看，也要从节约成本上来看——prompt caching策略。

从架构上看，一般prompt有system prompt、user prompt、memory
根据变化速度划分：
- Identity Layer：长期稳定——告知llm你是谁你的行动规则
- World State Layer：会话级稳定——工作环境工作条件工作进度工作目标等等
- Execution Layer：每轮变化——当前task、turn上下文、工具返回等

好处是什么？
1.好缓存，节约成本。分层化设计的Identity Layer命中率高，我们也可以在runtime里面去维护Identity pormpt的长期稳定，多命中（Anthropic的prompt caching策略）。
2.好调试——说话风格有问题找Identity，工作目标理解不对找World State，执行loop判断失误，找Execution。优化prompt也可以从这三个角度进行。
3.短期错误不会混入长期错误——如果没有层级边界，短期观察很容易混进长期设定

### Point3  Provider怎么适配？
先把前提介绍一下
为什么一定要有 provider 层？
外部协议不一样。
比如最常见的区别：

- OpenAI 常见的是 message list、role、tool calling 那一套
- Anthropic 更偏 content blocks、tool use blocks、cache control
- 不同 provider 对 system prompt 放哪、tool message 怎么表示、结构化输出怎么接，都不完全一样

如果你不单独抽一个 provider 层去消化这些差异，那这些协议细节就会一路污染到上层prompt builder
所以 provider 层到底是在干什么？
翻译官的工作

介绍完了前提，那么provider层如何构建？
一个好的 prompt system，不该直接长成“OpenAI prompt”或者“Anthropic prompt”。
它应该先长成：
**系统内部自己的 prompt 语义结构**
然后再通过 provider 层，分别翻译成：
- OpenAI 能吃的格式
- Anthropic 能吃的格式
- 其他模型能吃的格式
所以 provider 层存在的根本原因不是“为了支持多个模型看起来很高级”，  
而是：
> **为了保护上层语义不被下层协议污染。**

比如不管 provider 返回的是 function calling、tool use block 还是别的格式，  
都应该在上层被规约成统一的 assistant message / tool call result 结构。

举例：OpenAI adapter 怎么做
OpenAI 的 Chat Completions 文档里，核心输入是 `messages` 列表，也支持 `tools` 列表；模型返回里会有 `message`，结束原因里也可能出现 `tool_calls`

``` typescript
type OpenAIMessage = {
  role: "system" | "developer" | "user" | "assistant" | "tool";
  content?: string;
  tool_call_id?: string;
  tool_calls?: Array<{
    id: string;
    type: "function";
    function: {
      name: string;
      arguments: string;
    };
  }>;
};

type OpenAIPayload = {
  model: string;
  messages: OpenAIMessage[];
  tools?: Array<{
    type: "function";
    function: {
      name: string;
      description: string;
      parameters: Record<string, unknown>;
    };
  }>;
  tool_choice?: "auto" | "none" | "required";
  temperature?: number;
  max_tokens?: number;
};

export class OpenAIRenderer implements ProviderRenderer<OpenAIPayload> {
  renderRequest(request: NormalizedModelRequest): OpenAIPayload {
    return {
      model: request.model,
      messages: request.messages.flatMap((msg) => {
        if (msg.role === "tool") {
          return msg.parts
            .filter((p): p is ToolResultPart => p.type === "tool-result")
            .map((part) => ({
              role: "tool" as const,
              tool_call_id: part.toolCallId,
              content: part.resultJson,
            }));
        }

        if (msg.role === "assistant") {
          const text = msg.parts
            .filter((p): p is TextPart => p.type === "text")
            .map((p) => p.text)
            .join("\n");

          const toolCalls = msg.parts
            .filter((p): p is ToolCallPart => p.type === "tool-call")
            .map((p) => ({
              id: p.toolCallId,
              type: "function" as const,
              function: {
                name: p.name,
                arguments: p.argumentsJson,
              },
            }));

          return [{
            role: "assistant" as const,
            content: text || undefined,
            tool_calls: toolCalls.length ? toolCalls : undefined,
          }];
        }

        const text = msg.parts
          .filter((p): p is TextPart => p.type === "text")
          .map((p) => p.text)
          .join("\n");

        return [{
          role: msg.role as "system" | "developer" | "user",
          content: text,
        }];
      }),
      tools: request.tools?.map((tool) => ({
        type: "function",
        function: {
          name: tool.name,
          description: tool.description,
          parameters: tool.inputSchema,
        },
      })),
      tool_choice: request.toolChoice,
      temperature: request.temperature,
      max_tokens: request.maxOutputTokens,
    };
  }
}

```
### Point4 prompt里也要有should not逻辑
这一点是从 OpenAI Harmony 风格里学到的。
对于agent上下文的构建，很多地方都是共通的。比如你写skill的时候也要告诉agent“在什么情况下不应该使用这个skill”，其实prompt里也是。
只有“You should not/must not”才能真正给agent构建行为边界
告诉模型：
- 不该做什么
- 什么情况下不要继续
- 什么不属于你的职责
- 什么情况下应该停止猜测、转为澄清或上报
- 哪些行为虽然“看起来有帮助”，但其实越权了

should not 常见应该写什么？
1. 非目标约束
- 不要为了完成局部任务而改动无关模块
- 不要为了让结果看起来更好而引入未经验证的假设
- 不要把临时观察写成长期事实
2. 停止条件
- 证据不足时不要假装确认
- 遇到权限边界不要绕过
- 上下文接近容量阈值时不要继续硬做
- 修复流程若会触发危险压缩，不要继续推进
3. 不允许的捷径
- 不要伪造工具结果
- 不要假装已经验证过
- 不要把未执行的步骤说成已完成
- 不要把“建议”表述成“事实”
4. 作用域边界
- 不要擅自改依赖
- 不要擅自改配置
- 不要擅自扩大任务范围
- 不要碰未授权路径
### Point5 Prompt 不是一锅炖，得有 role、source、冲突逻辑处理

prompt 里的信息，不只是“内容”，还有“来源”和“优先级”。
prompt不能当成文本来写！
例如常见role可以有：
- system：系统级长期规则
- developer / runtime：开发者或运行时注入的执行规则
- user：用户当前请求
- tool：工具返回的客观观察
- memory：历史经验 / 历史摘要
- assistant prior output：模型自己上一轮说过的话
如果不设置role，memory盖过tool，就是明目张胆制造幻觉。
**不是所有输入都同权**

source也是同理：
例如：
“这个文件已经修改成功了”
    如果来自 tool result：可信
    如果来自 assistant 自己猜测：不可信

有了以上评估标准后，剩下的就是在system prompt里去设定这套评价体系的标准
这其实说明一个更深的点：
> prompt engineering 不是写作文，  
> 而是在做“信息治理”。

你要治理的不只是文本内容，  而是：
- 谁说的
- 什么时候说的
- 可信度如何
- 与别的信息冲突时怎么办
- 哪些能进长期记忆，哪些只能留在当轮执行层

==一个好的 prompt system，本质上不是在写漂亮文本，而是在做分层、控边界、定来源、裁冲突、适配 provider 的上下文工程==