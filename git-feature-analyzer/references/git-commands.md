# Git 命令速查

本文件汇总分析流程中用到的 Git 命令、典型输出与注意事项。配合主 SKILL.md 第五步使用。

## 1. 仓库检查

### 仓库根目录

```bash
git rev-parse --show-toplevel
```

输出示例：

```text
/home/user/project
```

用途：确定分析根目录。

### 工作区状态

```bash
git status --short
```

典型输出：

```text
 M src/main.cpp          # 已修改，未暂存
A  src/new.cpp           # 已暂存
?? scratch/temp.txt      # 未跟踪
```

如果有任何输出，说明工作区有未提交修改。分析报告必须声明本次分析基于已提交历史，不包含未提交内容。

### 远程仓库

```bash
git remote -v
```

输出示例：

```text
origin  git@github.com:user/project.git (fetch)
origin  git@github.com:user/project.git (push)
```

记录远程名称与 URL，便于后续判断远程跟踪分支。

### 全部分支

```bash
git branch -a
```

输出示例：

```text
* main
  feature-x
  remotes/origin/main
  remotes/origin/feature-x
  remotes/origin/release-v1
```

处理规则：

1. 排除 `HEAD -> origin/main` 这样的符号引用。
2. 合并本地与远程中重复的分支引用。
3. 保留仅远程存在的分支。
4. 按用户配置排除不需要分析的分支。
5. 不要默认排除 `master`、`main` 等主干分支，除非用户明确要求。

## 2. 固定分支 HEAD

```bash
git rev-parse refs/heads/<branch>
git rev-parse refs/remotes/origin/<branch>
```

输出示例：

```text
a1b2c3d4e5f6789012345678901234567890abcd
```

记录每个分支的 HEAD SHA，后续所有分析都用 SHA 而非分支名，避免分析期间分支发生移动。

## 3. 确定基准 Commit

### 第一父提交策略

```bash
git rev-list --first-parent --before="2026-01-01 00:00:00" -n 1 <branch>
```

时间格式：`"YYYY-MM-DD HH:MM:SS"`，时区由本地时区决定；若需明确时区使用 `2026-01-01T00:00:00+08:00`。

输出示例：

```text
abc1234def56789012345678901234567890abcd
```

若该命令无输出，说明指定日期之前没有第一父提交。需要标记该分支为「无有效基准」，待人工确认，不得随意替换为其他 Commit。

### 同时获取基准时间

```bash
git rev-list --first-parent --before="2026-01-01 00:00:00" --pretty=format:"%H|%ad" --date=iso-strict -n 1 <branch>
```

输出示例：

```text
commit abc1234def56789012345678901234567890abcd
abc1234def56789012345678901234567890abcd|2025-12-31T18:30:00+08:00
```

### 分支创建时间晚于起始日期的特殊处理

```bash
# 获取分支第一个提交
git rev-list --first-parent --reverse <branch> | head -1

# 获取分支创建时间
git log --reverse --pretty=format:"%ad" --date=iso-strict <branch> | head -1
```

对该类分支：

- 不能把全部代码变化都视为项目独有功能。
- 需要分析分支从主干分叉的位置（merge-base），区分「继承主干」「独立开发」「合并引入」三种来源。
- 默认统计口径是「指定日期之后分支代码所体现的净功能增量」。

## 4. 提取代码差异

### 文件统计

```bash
git diff --stat <BASE_SHA> <HEAD_SHA>
```

输出示例：

```text
src/main.cpp | 50 ++++++++++++++++---------
src/new.cpp  | 120 +++++++++++++++++++++++++++++++
2 files changed, 145 insertions(+), 25 deletions(-)
```

### 文件变化列表

```bash
git diff --name-status <BASE_SHA> <HEAD_SHA>
```

输出示例：

```text
M       src/main.cpp          # 修改
A       src/new.cpp           # 新增
D       src/old.cpp           # 删除
R100    src/old.cpp -> src/renamed.cpp   # 重命名
```

### 详细代码差异（含重命名检测）

```bash
git diff --find-renames <BASE_SHA> <HEAD_SHA>
```

输出为 unified diff 格式，含 hunk 头与上下文。

### 第一父提交历史

```bash
git log \
  --first-parent \
  --since="2026-01-01 00:00:00" \
  --pretty=format:"%H|%P|%ad|%s" \
  --date=iso-strict \
  <branch>
```

输出示例：

```text
def5678...|abc1234...,1234567...|2026-01-05T10:00:00+08:00|新增缓存机制
1234567...|abc1234...|2026-01-03T14:30:00+08:00|修复并发问题
```

按需补充非第一父提交分析 Merge 引入的功能。

## 5. 识别 Merge

### 列出 Merge Commit

```bash
git log --merges --oneline <branch>
```

输出示例：

```text
a1b2c3d Merge branch 'feature/cache' into main
```

### 查看 Merge Commit 的父提交

```bash
git show --no-patch --pretty=raw <MERGE_COMMIT>
```

输出示例：

```text
commit a1b2c3d...
Merge: abc1234 def5678
Author: ...
Committer: ...
    Merge branch 'feature/cache' into main
```

`Merge:` 字段列出父提交。第一父为被合并分支（main），第二父为来源分支（feature/cache）。根据本分支角色确定哪个父是本分支历史延续。

### 分析 Merge 引入的代码变化

```bash
# 第二父（来源分支）相对于第一父的差异
git diff <FIRST_PARENT> <SECOND_PARENT>

# 合并后本分支相对于合并前的差异
git diff <FIRST_PARENT>^1 <MERGE_COMMIT>
```

注意：Merge Commit 本身不是独立功能，应分析其实际合并的代码变化是否构成本分支的功能。

## 6. 识别 Cherry-pick

### 基础检测

```bash
git log --cherry-mark --right-only --no-merges base...branch
```

`-`-` 标记表示该提交是另一侧某个提交的 Cherry-pick。

### 计算 Patch ID

```bash
git show <COMMIT_SHA> --pretty=format: | git patch-id --stable
```

输出示例：

```text
abcdef1234567890abcdef1234567890abcdef12 <COMMIT_SHA>
```

`<COMMIT_SHA>` 为原始 Commit SHA，前 40 位十六进制为 stable patch-id。

### 跨分支比对

```bash
# 取两个分支的提交及其 Patch ID
for sha in $(git rev-list base...branch --no-merges); do
  echo "$(git show $sha --pretty=format: | git patch-id --stable)"
done
```

对两侧分别运行后按 patch-id 排序比较。

### 判断规则

| 比较结果 | 结论 |
|---|---|
| Commit SHA 相同 | 同一个 Git 提交 |
| Patch ID 相同 | 代码补丁相同或等价 |
| Patch ID 不同 | 不能据此断定功能不同 |
| 不同 Commit 实现同一功能 | 需结合 AI 语义判断 |
| 一个 Commit 含多个功能 | 需结合代码 diff 拆分 |
| Merge Commit | 不能直接视为一个新增功能 |
| Revert 操作 | 结合最终代码状态判断 |

## 7. 时区处理

Git 命令中的日期参数遵循本地时区。常见做法：

```bash
# 使用带时区的 ISO 格式（推荐）
git rev-list --before="2026-01-01T00:00:00+08:00" ...

# 使用本地时间字符串
git rev-list --before="2026-01-01 00:00:00" ...
```

在报告中显式声明使用的时区，避免服务器与开发环境时区不一致导致基准定位错误。

## 8. 其他常用

### 查询文件历史

```bash
git log --follow --pretty=format:"%H|%ad|%s" --date=iso-strict -- <file>
```

`--follow` 用于跟踪重命名。

### 查询特定提交

```bash
git show --stat <COMMIT_SHA>
```

输出某次提交的修改文件与统计。

### 取两个分支的共同祖先

```bash
git merge-base <branch1> <branch2>
```

用于判断分支之间的继承关系。

### 当前分支

```bash
git symbolic-ref --short HEAD
```

仅在需要切换分支时使用。本 Skill 推荐不切换分支，直接通过 SHA 与分支名定位。
