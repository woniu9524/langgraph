# 如何自定义 Dockerfile

用户可以在从父 LangGraph 镜像导入后，添加一个额外的行数组来修改 Dockerfile。要做到这一点，只需修改你的 `langgraph.json` 文件，通过 `dockerfile_lines` 键传递你想要运行的命令。例如，如果我们想在图中使用 `Pillow`，就需要添加以下依赖：

```json
{
    "dependencies": ["."],
    "graphs": {
        "openai_agent": "./openai_agent.py:agent",
    },
    "env": "./.env",
    "dockerfile_lines": [
        "RUN apt-get update && apt-get install -y libjpeg-dev zlib1g-dev libpng-dev",
        "RUN pip install Pillow"
    ]
}
```

这将安装使用 Pillow（如果我们处理 `jpeg` 或 `png` 图像格式的话）所需的系统软件包。