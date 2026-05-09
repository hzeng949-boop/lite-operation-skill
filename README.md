# Lite Operation Skill 仓库说明

这个仓库用于存放 Lite / Palmnet 后台操作相关的 skill。

目前的设计原则是：
- `lite-operation-workflow` 是唯一总入口
- 任何 Lite 后台任务都先读取 `lite-operation-workflow`
- 再根据任务类型调用对应的子 skill
- 子 skill 只保留自己独有的规则，不重复写共通规则

## 仓库结构

当前仓库包含：

- `lite-operation-workflow`
  总工作流。负责共通规则、环境确认、门店确认、token 与 scope 检查、接口入口、验证标准，以及未来新 workflow 的命名和 guidance 规则。
- `lite-product-batch-management`
  负责分类与商品的批量建立、修改、检查。
- `lite-combos-creation`
  负责 combo 的建立、修改、检查。
- `lite-table-creation`
  负责桌台的批量建立、修改、检查。
- `lite-table-qrcode-export`
  负责导出指定门店的桌台二维码对应表，默认输出 `桌台名称` 与 URL。

## Agent 使用方式

### 总原则

任何 Lite 后台任务，都先读：

- `lite-operation-workflow/SKILL.md`

确认完共通规则后，再决定调用哪一个子 skill。

### 调用顺序

1. 先读取 `lite-operation-workflow`
2. 先确认三件事：
   - 环境
   - 门店
   - 修改内容
3. 检查当前后台上下文：
   - token
   - storeId
   - scope
4. 再调用对应子 skill

### 子 skill 选择规则

- 商品 / 分类任务：
  使用 `lite-product-batch-management`
- combo 任务：
  使用 `lite-combos-creation`
- 桌台任务：
  使用 `lite-table-creation`
- 桌台二维码导出任务：
  使用 `lite-table-qrcode-export`

### 新增 workflow 的规则

如果后面要新增新的 Lite 后台流程：

1. 先看现有 skill 能不能覆盖
2. 如果不能覆盖，再新增新的 workflow skill
3. 新 skill 必须先引用 `lite-operation-workflow`
4. 命名按照：
   - `lite-<domain>-<action>`
   - 批量类用 `lite-<domain>-batch-<action>`
   - 审查类用 `lite-<domain>-inspection` 或 `lite-<domain>-audit`
5. 如果有额外页面说明、字段说明、例外规则，统一放到：
   - `references/guidance.md`

## 使用者怎么提需求

为了减少来回确认，建议需求里直接写清楚：

- 目标环境
  - development 或 production
- 目标门店
- 要改什么
- 来源资料
  - 例如 Excel、商品清单、combo 结构、桌台编号区间

### 例子

商品任务：

```text
请在 production 的某门店批量建立这批商品，分类和价格都按附件执行。
```

combo 任务：

```text
请在 production 的某门店建立一个 combo，名称、价格、group 和商品清单如下。
```

桌台任务：

```text
请在 production 的某门店建立桌台：
SALON S1-S40
BARRA B1-B10
```

## 强制确认规则

任何会写入后台的动作，在提交前都必须再次确认：

- 环境
- 门店
- 修改内容

只要这三项有一项不明确，就不应该提交。

## 验证标准

任务完成不等于“已经提交”。

必须确认：
- 写入的是正确环境
- 写入的是正确门店
- 结果和需求一致
- 抽查验证通过
- 没有隐藏的不一致

## 维护规则

维护这套仓库时，遵循下面原则：

- 共通规则只写在 `lite-operation-workflow`
- 子 skill 只写自己的特殊规则
- 不重复复制大段共通说明
- 新 workflow 先命名，再决定是否真的需要单独拆一个 skill
