
# Ollama 模型管理与故障排除指南

## 一、 基础查询命令

在终端中，你可以通过以下命令管理和查看本地模型：

| 功能 | 命令 | 说明 |
| :--- | :--- | :--- |
| **列出已下载模型** | `ollama list` | 查看本地所有已安装的模型、大小和 ID |
| **查看运行状态** | `ollama ps` | 查看当前正在内存/显存中运行的模型 |
| **查看模型详情** | `ollama show <模型名>` | 查看特定模型的参数、模板和授权信息 |
| **查看 Modelfile** | `ollama show --modelfile <模型名>` | 查看该模型是如何定义的 |

---

## 二、 解决 Pull 模型报错

### 1. 错误现象
执行 `ollama pull bge-small-zh` 时提示：
> `Error: pull model manifest: file does not exist`

### 2. 错误原因
*   **名称不匹配**：Ollama 官方模型库（Library）中没有名为 `bge-small-zh` 的直接标签。
*   **库限制**：Ollama 官方库只收录了部分经过验证的模型，并非 HuggingFace 上所有的模型都能直接通过 `pull` 命令获取。

### 3. 解决方法
请改用 Ollama 官方库中存在的 BGE 系列模型。推荐使用 **BGE-M3**，它是目前最强的多语言（含中文）向量模型。

**执行以下命令：**
```bash
ollama pull bge-m3
```

---

## 三、 推荐的中文向量（Embedding）模型

如果你需要处理中文文档或进行 RAG（检索增强生成），建议使用以下在 Ollama 官方库中已上架的模型：

1.  **bge-m3 (推荐)**
    *   **特点**：支持多语言（80+种），对中文支持极佳，支持长文本。
    *   **命令**：`ollama pull bge-m3`
2.  **nomic-embed-text**
    *   **特点**：通用性强，性能均衡。
    *   **命令**：`ollama pull nomic-embed-text`
3.  **bge-large**
    *   **特点**：参数量稍大，精度较高。
    *   **命令**：`ollama pull bge-large`

---

## 四、 高级进阶：如何导入特定的第三方模型

如果你坚持要使用特定的 `bge-small-zh`（例如你已经有了 GGUF 文件），可以按照以下步骤手动导入：

1.  **准备 GGUF 文件**：从 HuggingFace 下载 `bge-small-zh` 的 `.gguf` 格式文件。
2.  **创建 Modelfile**：在本地创建一个名为 `Modelfile` 的文件，内容如下：
    ```dockerfile
    FROM /你的路径/bge-small-zh.gguf
    ```
3.  **注册模型**：
    ```bash
    ollama create bge-small-zh -f Modelfile
    ```
4.  **验证**：
    使用 `ollama list` 即可看到新创建的 `bge-small-zh`。

---

## 五、 常用维护命令回顾

*   **更新模型**：`ollama pull <模型名>` (重复执行 pull 即可更新)
*   **删除模型**：`ollama rm <模型名>`
*   **运行并对话**：`ollama run <模型名>`

---
**文档生成时间**：2024年
**适用环境**：Linux / Ubuntu / Debian / CentOS