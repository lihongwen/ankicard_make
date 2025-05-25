# LightRAG工作区管理功能改进

## 功能介绍

本次更新对LightRAG系统的工作区管理功能进行了完善，主要解决了文档与工作区之间的关联问题。通过在文档的元数据中添加工作区标识字段，实现了不同工作区之间文档的隔离和切换。

## 修改内容

### 1. 文档查询API改进

修改`documents`接口，添加工作区过滤功能，只返回当前活动工作区的文档。这确保用户在切换工作区后只看到该工作区下的文档。

```python
async def documents() -> DocsStatusesResponse:
    # 获取当前活动工作区
    from .workspace_routes import get_active_workspace
    active_workspace = get_active_workspace()
    logger.info(f"获取文档列表，当前活动工作区: {active_workspace}")
    
    # 只包含当前活动工作区的文档
    if doc_workspace == active_workspace:
        response.statuses[status].append(...)
```

### 2. 文档入队流程增强

修改`apipeline_enqueue_documents`方法，在文档元数据中添加当前工作区信息，确保每个文档都能正确关联到其所属工作区。

```python
# 3. 生成文档初始状态
active_workspace = get_active_workspace()
logger.info(f"文档入队，当前活动工作区: {active_workspace}")

new_docs: dict[str, Any] = {
    id_: {
        # ...其他字段...
        "metadata": {
            "workspace": active_workspace,  # 添加工作区信息到元数据
        },
    }
    for id_, content_data in contents.items()
}
```

### 3. 文档处理过程优化

修改`process_document`函数，确保在文档处理的各个阶段(PROCESSING、PROCESSED、FAILED)都保留文档的工作区信息，防止工作区信息丢失。

```python
await self.doc_status.upsert(
    {
        doc_id: {
            "status": DocStatus.PROCESSING,
            # ...其他字段...
            "metadata": {
                "workspace": getattr(status_doc, "metadata", {}).get("workspace", "default"),  # 保留工作区信息
            },
        }
    }
)
```

### 4. 文件上传接口优化

为各种文件上传接口(单文件、批量文件、文本)添加工作区信息处理，确保通过不同方式添加的文档都正确关联到当前工作区。

```python
# 获取当前活动工作区
from .workspace_routes import get_active_workspace
active_workspace = get_active_workspace()
logger.info(f"文件入队 {file_path.name}，当前活动工作区: {active_workspace}")
```

## 实现效果

1. 用户切换工作区后，只能看到当前工作区的文档，实现工作区之间的完全隔离
2. 文档在处理过程中正确保留工作区信息
3. 各种文档添加方式(包括文件上传、文本输入)都能正确关联到当前工作区
4. 提供了详细的日志，记录文档与工作区的关联情况

## 后续优化方向

1. 添加工作区文档统计功能，显示每个工作区的文档数量
2. 支持跨工作区文档复制功能
3. 在前端界面上增加工作区切换的视觉提示，明确当前工作环境 