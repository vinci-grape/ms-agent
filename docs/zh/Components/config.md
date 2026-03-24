---
slug: config
title: 配置与参数
description: Ms-Agent 配置与参数：类型配置、自定义代码、LLM配置、推理配置、system和query、callbacks、工具配置、内存压缩配置、其他、config_handler、命令行配置
---

# 配置与参数

MS-Agent使用一个yaml文件进行配置管理，通常这个文件被命名为`agent.yaml`，这样的设计使不同场景可以读取不同的配置文件。该文件具体包含的字段有：

## 类型配置

> 可选

```yaml
# type: codeagent
type: llmagent
```

标识本配置对应的agent类型，支持`llmagent`和`codeagent`两类。默认为`llmagent`。如果yaml中包含了code_file字段，则code_file优先生效。

## 自定义代码

> 可选，在需要自定义LLMAgent时使用

```yaml
code_file: custom_agent
```

可以使用一个外部agent类，该类需要继承自`LLMAgent`。可以复写其中的若干方法，如果code_file有值，则`type`字段不生效。

## LLM配置

> 必须存在

```yaml
llm:
  # 大模型服务backend
  service: modelscope
  # 模型id
  model: Qwen/Qwen3-235B-A22B-Instruct-2507
  # 模型api_key
  modelscope_api_key:
  # 模型base_url
  modelscope_base_url: https://api-inference.modelscope.cn/v1
```

## 推理配置

> 必须存在

```yaml
generation_config:
  # 下面的字段均为OpenAI sdk的标准参数，你也可以配置OpenAI支持的其他参数在这里。
  top_p: 0.6
  temperature: 0.2
  top_k: 20
  stream: true
  extra_body:
    enable_thinking: false
```

## system和query

> 可选，但推荐传入system

```yaml
prompt:
  # LLM system，如果不传递则使用默认的`you are a helpful assistant.`
  system:
  # LLM初始query，通常来说可以不使用
  query:
```

## callbacks

> 可选，推荐自定义callbacks

```yaml
callbacks:
  # 用户输入callback，该callback在assistant回复后自动等待用户输入
  - input_callback
```

## 工具配置

> 可选，推荐使用

```yaml
tools:
  # 工具名称
  file_system:
    # 是否是mcp
    mcp: false
    # 排除的function，可以为空
    exclude:
      - create_directory
      - write_file
  amap-maps:
    mcp: true
    type: sse
    url: https://mcp.api-inference.modelscope.net/xxx/sse
    exclude:
      - map_geo
```

支持的完整工具列表，以及自定义工具请参考[这里](./tools)

## 内存压缩配置

> 可选，用于长对话场景的上下文管理

```yaml
memory:
  # 上下文压缩器：基于token检测 + 工具输出裁剪 + LLM摘要
  context_compressor:
    context_limit: 128000      # 模型上下文窗口大小
    prune_protect: 40000       # 保护最近工具输出的token阈值
    prune_minimum: 20000       # 最小裁剪数量
    reserved_buffer: 20000     # 预留缓冲区
    enable_summary: true       # 是否启用LLM摘要
    summary_prompt: |          # 自定义摘要提示词（可选）
      Summarize this conversation...

  # 精炼压缩器：保留执行轨迹的结构化压缩
  refine_condenser:
    threshold: 60000           # 触发压缩的字符阈值
    system: ...                # 自定义压缩提示词（可选）

  # 代码压缩器：生成代码索引文件
  code_condenser:
    system: ...                # 自定义索引生成提示词（可选）
    code_wrapper: ['```', '```']  # 代码块标记
```

支持的压缩器类型：

| 类型 | 适用场景 | 压缩方式 |
|------|---------|---------|
| `context_compressor` | 通用长对话 | Token检测 + 工具裁剪 + LLM摘要 |
| `refine_condenser` | 需保留执行轨迹 | 结构化消息压缩（1:6压缩比） |
| `code_condenser` | 代码生成任务 | 生成代码索引JSON |

## 其他

> 可选，按需配置

```yaml
# 自动对话轮数，默认为20轮
max_chat_round: 9999

# 工具调用超时时间，单位秒
tool_call_timeout: 30000

# 输出artifact目录
output_dir: output

# 帮助信息，通常在运行错误后出现
help: |
  A commonly use config, try whatever you want!
```

## config_handler

为了便于在任务开始时对config进行定制化，MS-Agent构建了一个名为`ConfigLifecycleHandler`的机制。这是一个callback类，开发者可以在yaml文件中增加这样一个配置：

```yaml
handler: custom_handler
```

这代表和yaml文件同级有一个custom_handler.py文件，该文件的类继承自`ConfigLifecycleHandler`，分别有两个方法：

```python
    def task_begin(self, config: DictConfig, tag: str) -> DictConfig:
        return config

    def task_end(self, config: DictConfig, tag: str) -> DictConfig:
        return config
```

`task_begin`在LLMAgent类构造时生效，在该方法中可以对config进行一些修改。如果你的工作流中下游任务会继承上游的yaml配置，这个机制会有帮助。值得注意的是`tag`参数，该参数会传入当前LLMAgent的名字，方便分辨当前工作流的节点。


## 命令行配置

在yaml配置之外，MS-Agent还支持若干额外的命令行参数。

- query: 初始query，这个query的优先级高于yaml中的prompt.query
- config: 配置文件路径，支持modelscope model-id
- trust_remote_code: 是否信任外部代码。如果某个配置包含了一些外部代码，需要将这个参数置为true才会生效
- load_cache: 从历史messages继续对话。cache会被自动存储在`output`配置中。默认为`False`
- mcp_server_file: 可以读取一个外部的mcp工具配置，格式为：
    ```json
    {
      "mcpServers": {
        "amap-maps": {
          "type": "sse",
          "url": "https://mcp.api-inference.modelscope.net/..."
        }
      }
    }
    ```

> agent.yaml中的任意一个配置，都可以使用命令行传入新的值, 也支持从同名（大小写不敏感）环境变量中读取，例如`--llm.modelscope_api_key xxx-xxx`。
