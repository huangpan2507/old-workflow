# 删除 Item 表数据迁移说明

## 迁移目标

删除测试专用的 `item` 表及其相关代码。

## 迁移文件

**Revision:** `b353e48d277e_remove_item_table.py`
**Down Revision:** `k1234567890f` (add_organization_function_heads_table)
**Create Date:** 2025-11-03 09:39:59

## 迁移内容

### Upgrade（升级）

删除 `item` 表：

```python
op.drop_table('item')
```

### Downgrade（回滚）

如果需要在未来回滚，可以重新创建 `item` 表：

```python
op.create_table('item',
    sa.Column('description', sa.VARCHAR(length=255), autoincrement=False, nullable=True),
    sa.Column('title', sa.VARCHAR(length=255), autoincrement=False, nullable=False),
    sa.Column('id', sa.UUID(), autoincrement=False, nullable=False),
    sa.Column('owner_id', sa.UUID(), autoincrement=False, nullable=False),
    sa.ForeignKeyConstraint(['owner_id'], ['users.id'], name='item_owner_id_fkey', ondelete='CASCADE'),
    sa.PrimaryKeyConstraint('id', name='item_pkey')
)
```

## 已删除的代码

### 后端代码

1. **模型类** (`backend/app/models.py`):
   - `ItemBase`
   - `ItemCreate`
   - `ItemUpdate`
   - `Item`
   - `ItemPublic`
   - `ItemsPublic`

2. **API 路由** (`backend/app/api/v1/items.py`): 整个文件

3. **CRUD 函数** (`backend/app/crud.py`):
   - `create_item()` 函数

4. **路由注册** (`backend/app/api/v1/__init__.py`):
   - 移除了 `items_router` 的导入和注册

5. **测试文件**:
   - `backend/app/tests/utils/item.py`
   - `backend/app/tests/api/routes/test_items.py`

### 前端代码

1. **路由文件** (`frontend/src/routes/_layout/items.tsx`): 整个文件

2. **组件目录** (`frontend/src/components/Items/`):
   - `AddItem.tsx`
   - `EditItem.tsx`
   - `DeleteItem.tsx`

3. **公共组件** (`frontend/src/components/Common/ItemActionsMenu.tsx`): 整个文件

4. **路由树** (`frontend/src/routeTree.gen.ts`):
   - 移除了 `items` 路由的引用

## 执行状态

- ✅ 迁移文件已生成：`b353e48d277e_remove_item_table.py`
- ✅ 数据库迁移已执行成功
- ✅ 当前数据库版本：`b353e48d277e (head)`
- ✅ `item` 表已从数据库中删除（验证通过）

## 注意事项

1. **自动生成的前端客户端代码**: `frontend/src/client/*.gen.ts` 中的 Item 相关类型会在重新生成 OpenAPI 客户端时自动更新。

2. **数据备份**: 如果在生产环境中有重要的 item 数据，请在执行迁移前进行备份。

3. **回滚支持**: 如果需要回滚，可以使用 `alembic downgrade -1` 命令回滚到上一个版本。

## 验证命令

```bash
# 查看当前迁移版本
docker compose exec backend alembic current

# 验证 item 表是否已删除（应返回 False）
docker compose exec backend python -c "from app.core.database import engine; from sqlalchemy import inspect; inspector = inspect(engine); tables = inspector.get_table_names(); print('item table exists:', 'item' in tables)"
```

