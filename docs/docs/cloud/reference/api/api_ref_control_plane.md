# LangGraph Control Plane API Reference

LangGraph Control Plane API 用于以编程方式创建和管理 LangGraph Server 部署。例如，可以编排这些 API 来创建自定义 CI/CD 工作流。

点击 <a href="https://api.host.langchain.com/docs" target="_blank">此处</a> 查看 API 参考。

## Host

LangGraph Control Plane 托管的 Cloud SaaS 数据区域：

| US | EU |
|----|----|
| `https://api.host.langchain.com` | `https://eu.api.host.langchain.com` |

**注意**: LangGraph Platform 的自托管部署将拥有 LangGraph Control Plane 的自定义主机。

## Authentication

要与 LangGraph Control Plane API 进行身份验证，请将 `X-Api-Key` 头设置为有效的 LangSmith API 密钥。

示例 `curl` 命令：
```shell
curl --request GET \
  --url http://localhost:8124/v2/deployments \
  --header 'X-Api-Key: LANGSMITH_API_KEY'
```

## Versioning

每个端点路径都以版本号（例如 `v1`、`v2`）作为前缀。

## Quick Start

1. 调用 `POST /v2/deployments` 来创建一个新的 Deployment。响应体包含 Deployment ID (`id`) 和最新（也是第一个）修订版的 ID (`latest_revision_id`)。
1. 调用 `GET /v2/deployments/{deployment_id}` 来检索 Deployment。将 URL 中的 `deployment_id` 设置为 Deployment ID (`id`) 的值。
1. 调用 `GET /v2/deployments/{deployment_id}/revisions/{latest_revision_id}` 来轮询修订版的 `status`，直到 `status` 为 `DEPLOYED`。
1. 调用 `PATCH /v2/deployments/{deployment_id}` 来更新部署。

## Example Code
下面是演示如何编排 LangGraph Control Plane API 来创建、更新和删除部署的示例 Python 代码。
```python
import os
import time

import requests
from dotenv import load_dotenv


load_dotenv()

# required environment variables
CONTROL_PLANE_HOST = os.getenv("CONTROL_PLANE_HOST")
LANGSMITH_API_KEY = os.getenv("LANGSMITH_API_KEY")
INTEGRATION_ID = os.getenv("INTEGRATION_ID")
MAX_WAIT_TIME = 1800  # 30 mins


def get_headers() -> dict:
    """Return common headers for requests to LangGraph Control Plane API."""
    return {
        "X-Api-Key": LANGSMITH_API_KEY,
    }


def create_deployment() -> str:
    """Create deployment. Return deployment ID."""
    headers = get_headers()
    headers["Content-Type"] = "application/json"

    deployment_name = "my_deployment"

    request_body = {
        "name": deployment_name,
        "source": "github",
        "source_config": {
            "integration_id": INTEGRATION_ID,
            "repo_url": "https://github.com/langchain-ai/langgraph-example",
            "deployment_type": "dev",
            "build_on_push": False,
            "custom_url": None,
            "resource_spec": None,
        },
        "source_revision_config": {
            "repo_ref": "main",
            "langgraph_config_path": "langgraph.json",
            "image_uri": None,
        },
        "secrets": [
            {
                "name": "OPENAI_API_KEY",
                "value": "test_openai_api_key",
            },
            {
                "name": "ANTHROPIC_API_KEY",
                "value": "test_anthropic_api_key",
            },
            {
                "name": "TAVILY_API_KEY",
                "value": "test_tavily_api_key",
            },
        ],
    }

    response = requests.post(
        url=f"{CONTROL_PLANE_HOST}/v2/deployments",
        headers=headers,
        json=request_body,
    )

    if response.status_code != 201:
        raise Exception(f"Failed to create deployment: {response.text}")

    deployment_id = response.json()["id"]
    print(f"Created deployment {deployment_name} ({deployment_id})")
    return deployment_id


def get_deployment(deployment_id: str) -> dict:
    """Get deployment."""
    response = requests.get(
        url=f"{CONTROL_PLANE_HOST}/v2/deployments/{deployment_id}",
        headers=get_headers(),
    )

    if response.status_code != 200:
        raise Exception(f"Failed to get deployment ID {deployment_id}: {response.text}")

    return response.json()


def list_revisions(deployment_id: str) -> list[dict]:
    """List revisions.

    Return list is sorted by created_at in descending order (latest first).
    """
    response = requests.get(
        url=f"{CONTROL_PLANE_HOST}/v2/deployments/{deployment_id}/revisions",
        headers=get_headers(),
    )

    if response.status_code != 200:
        raise Exception(
            f"Failed to list revisions for deployment ID {deployment_id}: {response.text}"
        )

    return response.json()


def get_revision(
    deployment_id: str,
    revision_id: str,
) -> dict:
    """Get revision."""
    response = requests.get(
        url=f"{CONTROL_PLANE_HOST}/v2/deployments/{deployment_id}/revisions/{revision_id}",
        headers=get_headers(),
    )

    if response.status_code != 200:
        raise Exception(f"Failed to get revision ID {revision_id}: {response.text}")

    return response.json()


def patch_deployment(deployment_id: str) -> None:
    """Patch deployment."""
    headers = get_headers()
    headers["Content-Type"] = "application/json"

    response = requests.patch(
        url=f"{CONTROL_PLANE_HOST}/v2/deployments/{deployment_id}",
        headers=headers,
        json={
            "source_config": {
                "build_on_push": True,
            },
            "source_revision_config": {
                "repo_ref": "main",
                "langgraph_config_path": "langgraph.json",
            },
        },
    )

    if response.status_code != 200:
        raise Exception(f"Failed to patch deployment: {response.text}")

    print(f"Patched deployment ID {deployment_id}")


def wait_for_deployment(deployment_id: str, revision_id: str) -> None:
    """Wait for revision status to be DEPLOYED."""
    start_time = time.time()
    revision, status = None, None
    while time.time() - start_time < MAX_WAIT_TIME:
        revision = get_revision(deployment_id, revision_id)
        status = revision["status"]
        if status == "DEPLOYED":
            break
        elif "FAILED" in status:
            raise Exception(f"Revision ID {revision_id} failed: {revision}")

        print(f"Waiting for revision ID {revision_id} to be DEPLOYED...")
        time.sleep(60)

    if status != "DEPLOYED":
        raise Exception(
            f"Timeout waiting for revision ID {revision_id} to be DEPLOYED: {revision}"
        )


def delete_deployment(deployment_id: str) -> None:
    """Delete deployment."""
    response = requests.delete(
        url=f"{CONTROL_PLANE_HOST}/v2/deployments/{deployment_id}",
        headers=get_headers(),
    )

    if response.status_code != 204:
        raise Exception(
            f"Failed to delete deployment ID {deployment_id}: {response.text}"
        )

    print(f"Deployment ID {deployment_id} deleted")


if __name__ == "__main__":
    # create deployment and get the latest revision
    deployment_id = create_deployment()
    revisions = list_revisions(deployment_id)
    latest_revision = revisions["resources"][0]
    latest_revision_id = latest_revision["id"]

    # wait for latest revision to be DEPLOYED
    wait_for_deployment(deployment_id, latest_revision_id)

    # patch the deployment and get the latest revision
    patch_deployment(deployment_id)
    revisions = list_revisions(deployment_id)
    latest_revision = revisions["resources"][0]
    latest_revision_id = latest_revision["id"]

    # wait for latest revision to be DEPLOYED
    wait_for_deployment(deployment_id, latest_revision_id)

    # delete the deployment
    delete_deployment(deployment_id)
```