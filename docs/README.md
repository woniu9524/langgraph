# 设置

要设置构建文档的要求，您可以运行：

```bash
uv sync --group test
```

## 在本地提供文档

要在本地运行文档服务器，您可以运行：

```bash
make serve-docs
```

这将会在 [http://127.0.0.1:8000/langgraph/](http://127.0.0.1:8000/langgraph/) 启动文档服务器。

## 执行笔记本

如果您想自动执行所有笔记本，以模仿“运行笔记本”的 GHA，您可以运行：

```bash
python _scripts/prepare_notebooks_for_ci.py
./_scripts/execute_notebooks.sh
```

**注意**：如果您想在不运行 `%pip install` 单元的情况下运行笔记本，您可以运行：

```bash
python _scripts/prepare_notebooks_for_ci.py --comment-install-cells
./_scripts/execute_notebooks.sh
```

`prepare_notebooks_for_ci.py` 脚本将为笔记本中的每个单元添加 VCR 磁带上下文管理器，以便：
* 首次运行时，带有网络请求的单元将被录制到 VCR 磁带文件
* 随后运行时，带有网络请求的单元将从磁带中回放

## 添加新的笔记本

如果您添加了一个带有 API 请求的笔记本，**建议**录制网络请求以便后续回放。如果未这样做，笔记本运行器每次运行笔记本时都会发出 API 请求，这可能会很昂贵且缓慢。

要录制网络请求، 请确保首先运行 `prepare_notebooks_for_ci.py` 脚本。

然后，运行

```bash
jupyter execute <path_to_notebook>
```

笔记本执行后，您应该会在 `cassettes` 目录中看到新录制的 VCR 磁带，并丢弃更新的笔记本。

## 更新现有笔记本

如果您正在更新现有笔记本，请确保删除 `cassettes` 目录中该笔记本的任何现有磁带（每个磁带都以笔记本名称为前缀），然后按照“添加新笔记本”部分中的步骤进行操作。

要为笔记本删除磁带，您可以运行：

```bash
rm cassettes/<notebook_name>*
```