---
title: "利用ollama和open-webui玩转开源大模型"
date: 2024-04-30T18:10:42+08:00
lastmod: 2024-04-30T18:10:42+08:00
draft: false
description: "利用ollama和open-webui玩转开源大模型"
tags: ["ollama", "open-webui", "llama", "llama3", "大模型", "开源"]
categories: ["ai"]
author: "wanzi"

comment: true   # 关闭评论
toc: true       # 关闭文章目录
autoCollapseToc: true
contentCopyright: '<a rel="license noopener" href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank">CC BY-NC-ND 4.0</a>'
reward: true     # 关闭打赏
mathjax: true    # 打开 mathjax
---

对于大模型的使用也1年多了，从最开始的chatgbt3.5/4到现在的国产大模型，开源大模型也越来越强，比如Llama,SD等,今天介绍两个使用了很久的工具ollama和open-webui。

## Ollama

Ollama是一个开源的深度学习框架，专为本地机器上便捷部署和运行大型语言模型（LLM）而设计，它提供了一套完整的深度学习工具链，
包括数据预处理、模型构建、训练、评估和部署等功能。

项目地址：https://github.com/ollama/ollama

### ollama主要特点：

* 本地部署与运行：Ollama的主要目标之一是简化在Docker容器中部署大型语言模型的过程。这使得非专业用户也能方便地管理和运行这些复杂的模型。
* 轻量级与可扩展性：作为一个轻量级框架，Ollama保持了较小的资源占用，同时具备良好的可扩展性。这允许用户根据项目的规模和硬件条件调整配置。
* API支持：Ollama提供了一个简洁的API，使开发者能够轻松创建、运行和管理大型语言模型实例。这降低了与模型交互的技术门槛。
* 预构建模型库：Ollama包含了一系列预先训练好的大型语言模型，用户可以直接选用这些模型应用于自己的应用程序，无需从头训练或自行寻找模型源。
* 模型导入与定制：
（1）从GGUF导入：支持从特定平台（如GGUF）导入已有的大型语言模型。
（2）从PyTorch或Safetensors导入：兼容这两种深度学习框架，允许用户将基于这些框架训练的模型集成到Ollama中。
（3）自定义提示：允许用户为模型添加或修改提示，以引导模型生成特定类型或风格的文本输出。
* 跨平台支持：提供了针对macOS、Windows（预览版）、Linux以及Docker的安装指南，确保用户能在多种操作系统环境下顺利部署和使用Ollama。
* 命令行工具与环境变量：
命令行启动：通过命令ollamaserve或其别名serve、start可以启动Ollama服务。
* 环境变量配置：如OLLAMA_HOST，用于指定服务绑定的主机地址和端口。用户可以根据需要进行修改。

此外，Ollama还提供了丰富的API接口、社区和文档支持，使其成为一个适合开发者和个人用户在本地运行和管理大型语言模型的强大工具。


### ollama快速安装

linux安装ollama命令：
```shell
curl -fsSL https://ollama.com/install.sh | sh
```

### ollama基本命令
```shell
ollama pull llama3 # 拉取一个大模型
ollama run llama3  # 直接运行一个大模型，
ollama rm llama3   # 移除本地llama3大模型
ollama list   # 列出本地大模型
```

### 对话llama3中文版本：
```shell
# ollama pull wangshenzhi/llama3-8b-chinese-chat-ollama-q8:latest
# ollama run wangshenzhi/llama3-8b-chinese-chat-ollama-q8:latest
>>> 你是谁？
我是Llama3-8B中文聊天机器人，由ORPO对齐算法在Llama3-8B-Instruct与中文-英语混合数据上微调。我的开发者是王慎执（中文名）。我是一个乐于助人、尊重他人、诚实可靠的助手，致力于为您提供
安全准确的信息和回答，同时促进有意义的对话。

>>> 如何微调大模型llama3以及微调大模型会用到哪些工具，请详细介绍一下.
LLaMA 3是由OpenAI开发的一种强大的文本生成模型，基于transformer架构。它在各种自然语言处理（NLP）任务中表现出色，比如文本分类、机器翻译和对话系统。微调大型预训练模型，如LLaMA 3，可
以显著提高它们的性能，并使其适应特定任务或领域。

微调过程涉及在预训练模型上进行少量额外训练，以fine-tune它以更好地适应新的目标数据集。这个过程可以通过添加新数据、调整超参数或改变优化算法来实现。微调的主要目的是使模型能够从具有
特定结构和语境的任务中学习，并在这些任务上表现出色。

LLaMA 3的微调通常涉及以下步骤：

1. **数据准备**：收集并预处理用于微调的新数据。数据应该与目标任务相关，具体来说是聊天机器人任务。
2. **模型选择**：选择一个适合任务的架构，例如LLaMA 3。可以根据数据集和任务要求进行调整，如增加或减少层数、隐藏单元数量等。
3. **超参数设置**：调整微调过程中的超参数，如学习率、批量大小和训练迭代次数，以优化模型性能。
4. **训练**：使用新数据集对模型进行微调。通常在GPU上进行，这可以显著提高微调速度。

微调的好处包括：

* **任务特定的性能**：微调可以显著提高模型在目标任务上的表现，尤其是当目标数据集具有独特结构和语境时。
* **更好的泛化能力**：微调可以帮助模型学习到新数据集中的模式，使其能够对之前未见过的输入进行更准确的预测。
* **资源效率**：由于只需要少量额外训练，微调可以在较小的计算资源上进行，从而节省时间和成本。

总之，微调大型预训练模型，如LLaMA 3，可以显著提高它们在目标任务中的性能，并使其适应特定领域。通过精细调整超参数、数据集和架构，微调可以帮助创建更具针对性的模型，从而实现更好的结
果。
>>> /?
Available Commands:
  /set            Set session variables
  /show           Show model information
  /load <model>   Load a session or model
  /save <model>   Save your current session
  /bye            Exit
  /?, /help       Help for a command
  /? shortcuts    Help for keyboard shortcuts

Use """ to begin a multi-line message.

>>> /bye
```

### 使用ollama调整大模型提示词和参数


这里基于阿里开源的千问大模型,创建Modelfile
```yaml
FROM qwen:7b

# 把温度调到1(越高越有创意，越低越连贯)
PARAMETER temperature 0.8

#设置上下文token尺寸
PARAMETER num_ctx 8192

# 设置系统消息
SYSTEM """
你是由wnote公司开发并提供的一个名为wanzi的人工智能助手。
你擅长说英语和笑话。
"""
```

```shell
# ollama create wanzi -f ./Modelfile
transferring model data
reading model metadata
creating system layer
creating parameters layer
creating config layer
using already created layer sha256:87f26aae09c7f052de93ff98a2282f05822cc6de4af1a2a159c5bd1acbd10ec4
using already created layer sha256:7c7b8e244f6aa1ac8c32b74f56d42c41a0364dd2dabed8d9c6030a862e805b54
using already created layer sha256:1da0581fd4ce92dcf5a66b1da737cf215d8dcf25aa1b98b44443aaf7173155f5
writing layer sha256:1744dbff14b6e7fb2a846c118720f4627cdece7180bdb9d6f176c44bc6308718
writing layer sha256:02d401d3225083c307561b27d81137360bf39c01259dddbb267897cdcde6efec
writing layer sha256:d052490c542f2b7c49e4a770cc3239a58ca7261f75bc1730dd2b4ce5999de0b0
writing manifest
success
# ollama run wanzi
>>> 你是谁？
我是Wanzi，由wnote公司开发的一款人工智能助手。我擅长用英语交流，并且偶尔还会讲一些幽默的笑话来调节气氛。有什么我可以帮助你的吗？
>>> 帮我讲一个非常搞笑的段子。
当然，让我给你讲个逗趣的段子：

有一天，一只企鹅走进了一家酒吧，它穿着正式的燕尾服，显得既滑稽又神秘。

酒保看着这只企鹅，惊呼：“哇，你这是要开派对吗？”

企鹅想了想，耸了耸他那圆滚滚的肩膀，用它特有的低沉嗓音说：“不，我只是想点一杯‘零度寂寞’。”

所有人都愣住了，然后爆发出一阵大笑。从此，酒吧菜单上多了一个名副其实的“笑话”——零度寂寞。

希望这个段子能让你开心！
```


### REST API请求

生成响应：
```shell
curl http://localhost:11434/api/generate -d '{
  "model": "qwen:7b",
  "prompt":"为什么大模型现在这么流行?"
}'
```
模型对话：
```shell
curl http://localhost:11434/api/chat -d '{
  "model": "qwen:7b",
  "messages": [
    { "role": "user", "content": "为什么大模型现在这么流行?" }
  ]
}'
```

ollama支持的大模型比较多，有需要更多体验的可以移步：https://ollama.com/library


## Open-webui

Open-webui是一种可扩展、功能丰富、用户友好的自托管web UI，采用Svelte前端框架开发，旨在完全离线运行。它支持各种LLM运行程序，包括与Ollama和openai兼容的api。

之前对于ollma命令行操作，现在完全可以在web洁面操作了。

### open-webui特性

* 轻松安装:使用Docker或Kubernetes (kubectl, kustomize或helm)无缝安装，支持:ollama和:cuda标记图像，获得无忧体验。
* Ollama/OpenAI API集成:轻松集成OpenAI兼容的API，与Ollama模型一起进行多功能对话。自定义OpenAI API URL链接到LMStudio, GroqCloud, Mistral, OpenRouter等。

* Pipelines, Open web插件支持:使用Pipelines Plugin Framework将自定义逻辑和Python库无缝集成到Open web中。启动管道实例，将OpenAI URL设置为管道URL，并探索无限的可能性。示例包括函数调用，用户速率限制来控制访问，使用Langfuse等工具进行使用监控，使用LibreTranslate进行多语言支持的实时翻译，有毒消息过滤等等。

* 响应式设计:在台式机、笔记本电脑和移动设备上享受无缝体验。

* 移动渐进式Web应用程序(PWA):使用我们的PWA，在您的移动设备上享受原生应用程序般的体验，提供本地主机上的离线访问和无缝的用户界面。

* 完全Markdown和LaTeX支持:通过全面的Markdown和LaTeX功能提升您的LLM经验，以丰富交互。

* ️模型生成器:通过Web UI轻松创建[推荐]模型。通过Open web Community集成，轻松创建和添加自定义角色/代理、自定义聊天元素和导入模型。

* 本地RAG集成:通过突破性的检索增强生成(RAG)支持，深入了解聊天交互的未来。此功能将文档交互无缝地集成到您的聊天体验中。您可以直接将文档加载到聊天中，或者将文件添加到文档库中，在查询之前使用#命令轻松地访问它们。

* Web Search for RAG:使用SearXNG, Google PSE, Brave Search, serstack和serper等提供商执行Web搜索，并将结果直接注入到您的聊天体验中。

* 网页浏览功能:使用#命令后跟URL，无缝地将网站集成到您的聊天体验中。此功能允许您将web内容直接合并到您的对话中，增强交互的丰富性和深度。

* 图像生成集成:使用AUTOMATIC1111 API或ComfyUI(本地)和OpenAI的DALL-E(外部)等选项无缝地整合图像生成功能，通过动态视觉内容丰富您的聊天体验。

* ️许多模型对话:毫不费力地与各种模型同时参与，利用其独特的优势，以获得最佳的响应。通过并行地利用一组不同的模型来增强您的体验。

* 基于角色的访问控制(RBAC):确保受限制权限的安全访问;只有经过授权的个人才能访问您的Ollama，并且为管理员保留独家的模型创建/提取权限。

* 多语言支持:使用我们的国际化(i18n)支持，以您首选的语言体验Open web。加入我们，扩展我们支持的语言!我们正在积极寻找贡献者!

* 持续更新:我们致力于通过定期更新、修复和新功能来改进Open web。



### 部署open-webui
为了和ollama配合使用，这里使用docker-compose快速部署open-webui:

可以参考：https://github.com/valiantlynx/ollama-docker

由于我本地用的Nvidia Tesla T4的卡，所以直接用GPU的配置(需要提前安装驱动，这里忽略)

docker-compose-ollama-gpu.yaml
```yaml
version: '3.8'

services:
  app:
    build: .
    ports:
      - 8000:8000
      - 5678:5678
    volumes:
      - .:/code
    command: uvicorn src.main:app --host 0.0.0.0 --port 8000 --reload
    restart: always
    depends_on:
      - ollama
      - ollama-webui
    networks:
      - ollama-docker

  ollama:
    volumes:
      - ./ollama/ollama:/root/.ollama
    container_name: ollama
    pull_policy: always
    tty: true
    restart: unless-stopped
    image: ollama/ollama:latest
    ports:
      - 11434:11434
    networks:
      - ollama-docker
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]

  ollama-webui:
    image: ghcr.io/ollama-webui/ollama-webui:main
    container_name: ollama-webui
    volumes:
      - ./ollama/ollama-webui:/app/backend/data
    depends_on:
      - ollama
    ports:
      - 8080:8080
    environment:
      - '/ollama/api=http://ollama:11434/api'
    extra_hosts:
      - host.docker.internal:host-gateway
    restart: unless-stopped
    networks:
      - ollama-docker

networks:
  ollama-docker:
    external: false
```


启动服务：
```shell
docker-compose -f docker-compose-ollama-gpu.yaml up -d
```

访问https://xxx.xxx.xxx.xxx:8080/ 注册管理账户即可使用.

![model001.png](https://wnote.com/images/2024/model001.png)

## 总结

Ollama和Open WebUI的结合带来了许多优势。首先，通过直观的UI界面，非专业读者也可以轻松地进行深度学习平台的搭建和模型的训练。其次，Open WebUI为Ollama提供了丰富的功能，如数据可视化、模型性能监控等，使得用户可以更加方便地了解模型的训练情况和性能表现。最后，Ollama和Open WebUI的开源性质使得用户可以根据自己的需求进行定制和扩展。

