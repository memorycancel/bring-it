---
title: 在本地部署 DeepSeek 大模型
layout: home
---

# 在本地部署 DeepSeek 大模型

2025-03-04 03：00

## 1 环境准备

### 1.1 安装mise、Python和CUDA

确保已安装 Python 3.8+ 和 CUDA 11.8+（如需GPU支持）。

```shell
# 安装 mise 和 python（略）
# 安装 cuda 参考：https://docs.nvidia.com/cuda/cuda-installation-guide-linux/#network-repo-installation-for-ubuntu
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt-get update
sudo apt-get install -y cuda-toolkit
sudo apt-get install -y nvidia-gds
sudo reboot
```

### 1.2 安装PyTorch和依赖库

```bash
pip install -r https://github.com/deepseek-ai/DeepSeek-Coder/blob/main/requirements.txt
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118  # CUDA 11.8
pip install transformers accelerate bitsandbytes sentencepiece
```
### 1.3 获取模型权重

从Hugging Face下载。先安装 git-lfs。使用git-lfs克隆仓库：

```bash
git lfs install
git clone https://huggingface.co/deepseek-ai/deepseek-coder-6.7b-instruct
```
### 1.4 编写推理代码

创建 inference.py 文件。参考 README：

```python
from transformers import AutoTokenizer, AutoModelForCausalLM
import torch

tokenizer = AutoTokenizer.from_pretrained("deepseek-coder-6.7b-instruct", trust_remote_code=True)

# model = AutoModelForCausalLM.from_pretrained("deepseek-coder-6.7b-instruct", trust_remote_code=True, torch_dtype=torch.bfloat16).cuda()
model = AutoModelForCausalLM.from_pretrained("deepseek-coder-6.7b-instruct", trust_remote_code=True, torch_dtype=torch.bfloat16).cpu()

messages=[
    { 'role': 'user', 'content': "用 ruby 写一个插入排序"}
]

inputs = tokenizer.apply_chat_template(messages, add_generation_prompt=True, return_tensors="pt").to(model.device)
# tokenizer.eos_token_id is the id of <|EOT|> token

outputs = model.generate(inputs, max_new_tokens=512, do_sample=False, top_k=50, top_p=0.95, num_return_sequences=1, eos_token_id=tokenizer.eos_token_id)

print(tokenizer.decode(outputs[0][len(inputs[0]):], skip_special_tokens=True))
```

运行模型
```bash
python3 inference.py
```

## 2 常见问题处理

### 2.1 显存不足

4-bit量化加载：

```python
model = AutoModelForCausalLM.from_pretrained(
    model_path,
    load_in_4bit=True,
    device_map="auto",
)
```
仅使用CPU（不推荐，速度慢）

```python
model = AutoModelForCausalLM.from_pretrained(model_path, device_map="cpu")
```
附注
模型大小与显存需求：

7B模型：约需16GB显存（FP16），4-bit量化后约6GB。

### 2.2 模型许可证

确保遵守DeepSeek模型的使用协议（如禁止商用或需申请许可）。

## 3 关于模型大小

### 3.1 模型有多大？

+ 参数规模：6.7B（67亿参数），属于“中等规模”大语言模型。
+ 实际存储占用：如果以 FP16（半精度） 格式存储，模型文件约 13-14GB。

下载后的模型文件夹通常包含权重、配置、分词器文件，总大小约 25-30GB（因元数据和版本差异略有不同）。

{: .note :}
总大小粗略计算公式： 6.7(B) x 4 = 26.8 （GB）

{: .note :}
以 FP16（半精度） 格式存储模型文件粗略计算公式： 6.7(B) x 2 = 13.4 （GB）

{: .note :}
权重、配置、分词器文件小粗略计算公式： 6.7(B) x 2 = 13.4 （GB）

### 3.2 为什么这么大？

#### 原因 1：参数规模与性能平衡

代码生成任务需要模型理解复杂语法、逻辑和长上下文依赖，较大的参数量（如 6.7B）能更好捕捉这些模式。
相比更小的模型（如 1B），6.7B 模型在代码生成质量、多语言支持上显著提升。

#### 原因 2：训练数据多样性

训练数据包含多编程语言代码（Python、Java、C++等）和跨领域文本（中英文文档、Stack Overflow 等），需要大容量模型存储多样知识。

#### 原因 3：行业趋势

代码模型（如 CodeLlama 7B/13B、StarCoder 15B）普遍向更大规模发展，以提升生成代码的准确性和复杂性支持。

{: .note :}
附加说明：模型大越大硬件需求越高：FP16 推理需 16GB 以上显存（如 NVIDIA 3090/4090）。
4-bit 量化后可降至 6-8GB 显存（使用 bitsandbytes 库）。

### 3.3 推理速度：

6.7B 模型在消费级 GPU 上生成代码的速度约为 5-15 token/秒（依赖硬件和生成长度）。

### 3.4 总结建议

用途匹配：如果专注代码生成/补全，6.7B 是性价比之选；若需更强的中文对话能力，可考虑混合使用专用中文模型。

资源优化：通过量化（4/8-bit）或模型切分（如 accelerate 库）降低显存占用。

官方文档：访问 [DeepSeek-Coder Hugging Face 页面](https://huggingface.co/deepseek-ai) 获取最新信息。

## 4 DeepSeek 模型的命名规则

### 4.1 核心命名结构

{: .note :}
deepseek-{领域}-{参数规模}-{版本/用途}

{: .important :}
领域：coder（代码）、llm（通用语言模型）、math（数学）、r1（全能型）。

参数规模：

+ 1.3b = 13亿参数
+ 6.7b = 67亿参数
+ 7b = 70亿参数
+ 560b = 5600亿参数（需超算级硬件）

版本/用途：

+ base：基础预训练模型（需微调后使用）。
+ instruct / chat：指令微调版（直接交互）。
+ v1.5 / r1：版本迭代标记（如改进训练数据或架构）。

### 4.2 命名示例解析

### 示例 1：deepseek-coder-33b-instruct-v1.5

+ 领域：代码模型
+ 参数：330亿
+ 用途：指令微调版
+ 版本：第 1.5 次迭代（改进代码生成质量）

### 示例 2：deepseek-llm-7b-chat

+ 领域：通用对话
+ 参数：70亿
+ 用途：聊天优化版

### 4.3 模型选择建议

### 根据需求匹配模型类型

+ 代码生成	deepseek-coder-6.7b-instruct
+ 通用问答	deepseek-llm-7b-chat
+ 数学/公式推导	deepseek-math-7b-r1
+ 企业级大规模应用	deepseek-r1-560b（需申请权限）

### 根据硬件选择参数规模

+ 消费级 GPU（16G显存）	≤7B（4-bit量化）
+ 多卡服务器	33B~70B
+ 超算集群	560B

### 4.4 附加说明

+ 版本迭代：模型后缀如 v1.5、r1 表示训练数据或架构改进，建议优先选择更高版本。
+ 许可证限制：部分模型（如 r1-560b）需申请商用许可，代码模型需遵守禁止生成恶意代码的协议。
+ 中文支持：除代码模型外，deepseek-llm 和 deepseek-r1 的中文能力更强，适合多轮对话。

如需最新模型列表，建议访问官方仓库：

+ Hugging Face: [https://huggingface.co/deepseek-ai](https://huggingface.co/deepseek-ai)
+ 官方文档: [https://deepseek.com/docs](https://deepseek.com/docs)