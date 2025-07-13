# LangGraph CLI

LangGraph 的官方命令行界面，提供创建、开发和部署 LangGraph 应用程序的工具。

## 安装

通过 pip 安装：
```bash
pip install langgraph-cli
```

用于开发模式并支持热重载：
```bash
pip install "langgraph-cli[inmem]"
```

## 命令

### `langgraph new` 🌱
从模板创建新的 LangGraph 项目
```bash
langgraph new [PATH] --template TEMPLATE_NAME
```

### `langgraph dev` 🏃‍♀️
在开发模式下运行 LangGraph API 服务器，支持热重载
```bash
langgraph dev [OPTIONS]
  --host TEXT                 绑定到的主机 (默认: 127.0.0.1)
  --port INTEGER             绑定到的端口 (默认: 2024)
  --no-reload               禁用自动重载
  --debug-port INTEGER      启用远程调试
  --no-browser             跳过打开浏览器窗口
  -c, --config FILE        配置文件路径 (默认: langgraph.json)
```

### `langgraph up` 🚀
在 Docker 中启动 LangGraph API 服务器
```bash
langgraph up [OPTIONS]
  -p, --port INTEGER        要暴露的端口 (默认: 8123)
  --wait                   等待服务启动
  --watch                  文件更改时重启
  --verbose               显示详细日志
  -c, --config FILE       配置文件路径
  -d, --docker-compose    额外的服务文件
```

### `langgraph build`
构建 LangGraph 应用程序的 Docker 镜像
```bash
langgraph build -t IMAGE_TAG [OPTIONS]
  --platform TEXT          目标平台 (例如：linux/amd64,linux/arm64)
  --pull / --no-pull      使用最新版/本地基础镜像
  -c, --config FILE       配置文件路径
```

### `langgraph dockerfile`
为自定义部署生成 Dockerfile
```bash
langgraph dockerfile SAVE_PATH [OPTIONS]
  -c, --config FILE       配置文件路径
```

## 配置

CLI 使用 `langgraph.json` 配置文件，包含以下关键设置：

```json
{
  "dependencies": ["langchain_openai", "./your_package"],  // 必需：包依赖项
  "graphs": {
    "my_graph": "./your_package/file.py:graph"            // 必需：图定义
  },
  "env": "./.env",                                        // 可选：环境变量
  "python_version": "3.11",                               // 可选：Python 版本 (3.11/3.12)
  "pip_config_file": "./pip.conf",                        // 可选：pip 配置
  "dockerfile_lines": []                                  // 可选：额外的 Dockerfile 命令
}
```

有关详细配置选项，请参阅[完整文档](https://langchain-ai.github.io/langgraph/cloud/reference/cli/)。

## 开发

要自行开发 CLI：

1. 克隆仓库
2. 进入 CLI 目录：`cd libs/cli`
3. 安装开发依赖项：`uv pip install`
4. 对 CLI 代码进行修改
5. 测试您的更改：
   ```bash
   # 直接运行 CLI 命令
   uv run langgraph --help
   
   # 或者使用示例
   cd examples
   uv pip install
   uv run langgraph dev  # 或其他命令
   ```

## 许可证

本项目根据存储库 LICENSE 文件中指定的条款获得许可。