# 首次使用初始化指南

当 `references/local-config.md` 不存在时，说明这是第一次使用，按下面流程走。

## 前置条件

1. 本机已安装 `lark-cli` 并完成飞书 OAuth 登录（跑 `lark-cli auth status` 能看到已登录用户）
2. 用户有一张飞书多维表格用来存书。两种情况都支持：
   - **已经有表**：直接把表链接发来，agent 自动读取现有字段结构适配
   - **还没有表**：照 [field-template.md](field-template.md) 新建一张，再把链接发来

## 初始化步骤

### 1. 拿到表链接

让用户把他的飞书多维表格分享链接发来（形如 `https://xxx.feishu.cn/base/xxxxx`）。

> **必须提醒用户确认编辑权限**：光有链接不够，写入需要编辑权限。告诉用户：
> 1. 打开这个多维表格
> 2. 点右上角"分享"
> 3. 确认当前登录用户（或机器人）有"可编辑"权限，不是"可阅读"
> 4. 如果链接分享设置的是"仅协作者可访问"，需要把当前使用 lark-cli 登录的账号加为协作者
>
> 如果跳过这步，后面写入会报 403 / permission denied。

> 如果用户还没有表，引导他先建一个，字段可参考 field-template.md，但不必完全一致——后续步骤会自动适配他实际的字段名。

### 2. 解析 base_token

```bash
lark-cli base +url-resolve --url "<用户的表链接>" --as user
```

拿到 `base_token`。

### 3. 找到书籍表

```bash
lark-cli base +table-list --base_token "<base_token>" --as user
```

从返回的 tables 里找到用户存书的那张表（可能叫"书籍""书架""reading list"等），让用户确认一下，记下 `table_id`。

### 4. 拉字段列表，做语义映射并和用户确认

```bash
lark-cli base +field-list --base_token "<base_token>" --table_id "<table_id>" --as user
```

**不要要求用户改字段名去匹配模板。** 按下面三步走：

#### 4.1 先按字段名猜

根据字段名和类型，自动猜测每个字段对应什么语义角色：

| 语义角色 | 猜测线索 |
|---|---|
| 书名（必填） | 主字段/第一个 text 字段，名字含"书名/标题/title/name" |
| 作者 | text 字段，名字含"作者/作家/author" |
| 分类/标签 | select/multiselect 字段 |
| 出版日期 | datetime 字段，名字含"出版/publish" |
| 简介/内容 | text 字段，名字含"简介/摘要/description" |
| 阅读日期 | datetime 字段，名字含"日期/读完/加入/读/date" |
| 阅读状态 | select 字段，选项是待读/在读/读完这类状态 |
| 读书笔记 | text 字段，名字含"笔记/感想/心得/评论/review" |

#### 4.2 列出来给用户确认

把猜测的映射整理成一张表发给用户：

```
我读了你表里的字段，猜测对应关系如下：

| 你表里的字段名 | 我猜它是 | 类型 |
|---|---|---|
| 书名 | 书名 | text |
| 作者 | 作者 | text |
| 标签 | 分类/标签 | 多选 |
| 读完日期 | 阅读日期 | 日期 |
| 状态 | 阅读状态 | 单选 |
| 我的感想 | 读书笔记 | text |
| 页数 | 不确定，你要存吗？ | number |
| 评分 | 不确定，你要存吗？ | number |

哪几个对不上？或者哪个字段你希望我每次都填/不填？
```

#### 4.3 按用户反馈处理

- **字段对得上**：确认后写入映射
- **字段名和语义对不上**（比如叫"备注"的 text 其实是读书笔记）：按用户说的改
- **多余字段**（页数、评分这类读书用不上的）：问用户要不要每次录入时都填，还是跳过
- **缺必填字段**（没有书名字段）：告诉用户缺什么，问他要不要加，不要擅自建

#### 4.4 记下精确选项名

从 field-list 结果里，把分类字段和状态字段的所有选项名完整记下来——**包括空格、emoji、大小写**，后续写入必须逐字匹配。

### 5. 配置视图排序（可选但推荐）

如果表里有日期字段，把主视图设置成按日期倒序，新书自动排最上面：

```bash
lark-cli base +view-list --base_token "<base_token>" --table_id "<table_id>" --as user
# 拿到主视图 view_id 后：
lark-cli base +view-set-sort \
  --base_token "<base_token>" --table_id "<table_id>" --view_id "<view_id>" \
  --json '{"sort_config":[{"field":"<日期字段名>","desc":true}]}' --as user
```

### 6. 保存配置

把上面拿到的所有值写入 `references/local-config.md`，格式如下：

```markdown
# 本地配置（首次初始化时生成）

- base_token: <从步骤2拿到>
- table_id: <从步骤3拿到>
- view_id: <从步骤5拿到>
- 身份: --as user

## 字段映射（按用户实际字段名）
| 语义角色 | 用户表里的字段名 | field_id | 类型 |
|---|---|---|---|
| 书名 | | fldXXX | text |
| 作者 | | fldXXX | text |
| 分类/标签 | | fldXXX | select/multiselect |
| 出版日期 | | fldXXX | datetime |
| 简介 | | fldXXX | text |
| 阅读日期 | | fldXXX | datetime |
| 阅读状态 | | fldXXX | select |
| 读书笔记 | | fldXXX | text |
| <多余但用户要求填的字段> | | fldXXX | ... |

## 分类/标签选项
（从步骤4的 field-list 结果填入完整选项名列表）

## 状态选项
待读 / 在读 / 已读 对应的实际选项名
```

写完后告诉用户"初始化完成，以后直接说'记本书：XXX'就行"。

## 注意事项

- local-config.md 包含用户私人的表访问凭证，**不要分享给别人**
- 如果用户换了表或改了表结构，重新跑一遍初始化即可覆盖 local-config.md
- 如果 `lark-cli` 没登录，引导用户先跑 `lark-cli auth login`
- 适配已有表时，**用用户的字段名，不要让用户改表来适配 skill**
