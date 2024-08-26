# 变量赋值

定义

变量赋值节点用于向可写入变量进行变量赋值，已支持以下可写入变量：

* [会话变量](../key-concept.md#hui-hua-bian-liang)。

用法：通过变量赋值节点，你可以将工作流内的变量赋值到会话变量中用于临时存储，并可以在后续对话中持续引用。

<figure><img src="../../../../img/zh-conversation-variable.png" alt="" width="375"><figcaption></figcaption></figure>

***

### 使用场景示例

通过变量赋值节点，你可以将会话过程中的**上下文、上传至对话框的文件（即将上线）、用户所输入的偏好信息**等写入至会话变量，并在后续对话中引用已存储的信息导向不同的处理流程或者进行回复。

**场景 1**

**自动判断提取并存储对话中的信息**，在会话内通过会话变量数组记录用户输入的重要信息，并在后续对话中让 LLM 基于会话变量中存储的历史信息进行个性化回复。

示例：开始对话后，LLM 会自动判断用户输入是否包含需要记住的事实、偏好或历史记录。如果有，LLM 会先提取并存储这些信息，然后再用这些信息作为上下文来回答。如果没有新的信息需要保存，LLM 会直接使用自身的相关记忆知识来回答问题。

<figure><img src="../../../.gitbook/assets/中文.jpeg" alt=""><figcaption></figcaption></figure>

**配置流程：**

1. **设置会话变量**：首先设置一个会话变量数组 `memories`，类型为 array\[object]，用于存储用户的事实、偏好和历史记录。
2. **判断和提取记忆**：
   * 添加一个条件判断节点，使用 LLM 来判断用户输入是否包含需要记住的新信息。
   * 如果有新信息，走上分支，使用 LLM 节点提取这些信息。
   * 如果没有新信息，走下分支，直接使用现有记忆回答。
3. **变量赋值/写入**：
   * 在上分支中，使用变量赋值节点，将提取出的新信息追加（append）到 `memories` 数组中。
   * 使用转义功能将 LLM 输出的文本字符串转换为适合存储在 array\[object] 中的格式。
4. **变量读取和使用**：
   * 在后续的 LLM 节点中，将 `memories` 数组中的内容转换为字符串，并插入到 LLM 的提示词 Prompts 中作为上下文。
   * LLM 使用这些历史信息来生成个性化回复。

图中的 code 节点代码如下：

1. 将字符串转义为 object

```python
import json

def main(arg1: str) -> object:
    try:
        # Parse the input JSON string
        input_data = json.loads(arg1)
        
        # Extract the memory object
        memory = input_data.get("memory", {})
        
        # Construct the return object
        result = {
            "facts": memory.get("facts", []),
            "preferences": memory.get("preferences", []),
            "memories": memory.get("memories", [])
        }
        
        return {
            "mem": result
        }
    except json.JSONDecodeError:
        return {
            "result": "Error: Invalid JSON string"
        }
    except Exception as e:
        return {
            "result": f"Error: {str(e)}"
        }
```

2. 将 object 转义为字符串

```python
import json

def main(arg1: list) -> str:
    try:
        # Assume arg1[0] is the dictionary we need to process
        context = arg1[0] if arg1 else {}
        
        # Construct the memory object
        memory = {"memory": context}
        
        # Convert the object to a JSON string
        json_str = json.dumps(memory, ensure_ascii=False, indent=2)
        
        # Wrap the JSON string in <answer> tags
        result = f"<answer>{json_str}</answer>"
        
        return {
            "result": result
        }
    except Exception as e:
        return {
            "result": f"<answer>Error: {str(e)}</answer>"
        }
```

**场景 2**

**记录用户的初始偏好信息**，在会话内记住用户输入的语言偏好，在后续对话中持续使用该语言类型进行回复。

示例：用户在对话开始前，在 `language` 输入框内指定了 “中文”，该语言将会被写入会话变量，LLM 在后续进行答复时会参考会话变量中的信息，在后续对话中持续使用“中文”进行回复。

<figure><img src="../../../../img/zh-conversation-var-scenario-1.png" alt=""><figcaption></figcaption></figure>

**配置流程：**

**设置会话变量**：首先设置一个会话变量 `language`，在会话流程开始时添加一个条件判断节点，用来判断 `language` 变量的值是否为空。

**变量写入/赋值**：首轮对话开始时，若 `language` 变量值为空，则使用 LLM 节点来提取用户输入的语言，再通过变量赋值节点将该语言类型写入到会话变量 `language` 中。

**变量读取**：在后续对话轮次中 `language` 变量已存储用户语言偏好。在后续对话中，LLM 节点通过引用 language 变量，使用用户的偏好语言类型进行回复。

**场景 3**

**辅助 Checklist 检查**，在会话内通过会话变量记录用户的输入项，更新 Checklist 中的内容，并在后续对话中检查遗漏项。

示例：开始对话后，LLM 会要求用户在对话框内输入 Checklist 所涉及的事项，用户一旦提及了 Checklist 中的内容，将会更新并存储至会话变量内。LLM 会在每轮对话后提醒用户继续补充遗漏项。

<figure><img src="../../../../img/conversation-var-scenario-2-1.png" alt=""><figcaption></figcaption></figure>

**配置流程：**

* **设置会话变量：** 首先设置一个会话变量 `ai_checklist`，在 LLM 内引用该变量作为上下文进行检查。
* **变量赋值/写入：** 每一轮对话时，在 LLM 节点内检查 `ai_checklist` 内的值并比对用户输入，若用户提供了新的信息，则更新 Checklist 并将输出内容通过变量赋值节点写入到 `ai_checklist` 内。
* **变量读取：** 每一轮对话读取 `ai_cheklist` 内的值并比对用户输入直至所有 checklist 完成。

***

### 使用变量赋值节点

点击节点右侧 ＋ 号，选择“变量赋值”节点，填写“赋值的变量”和“设置变量”。

<figure><img src="../../../../img/zh-language-variable-assigner.png" alt="" width="375"><figcaption></figcaption></figure>

**设置变量：**

赋值的变量：选择被赋值变量，即指定需要被赋值的目标会话变量。

设置变量：选择需要赋值的变量，即指定需要被转换的源变量。

以上图赋值逻辑为例：将上一个节点的文本输出项 `Language Recognition/text` 赋值到会话变量 `language` 内。

**写入模式：**

* 覆盖，将源变量的内容覆盖至目标会话变量
* 追加，指定变量为 Array 类型时
* 清空，清空目标会话变量中的内容