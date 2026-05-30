---
name: push-to-github
description: "将 Qoder skill 或任意本地目录快速推送到 GitHub 仓库。使用 git clone + copy + push 方式，避免逐文件 API 调用，速度极快。触发：推送GitHub、上传GitHub、push github、传到github、同步到github、skill上传。"
---

# Push to GitHub — 快速推送本地内容到 GitHub

使用 `git clone + copy + push` 流程，一次推送所有文件，比逐文件 API 上传快数倍。

## 核心原则

- **不逐文件调 API**，用 git 本地操作一次推送
- **不硬编码 token**，依赖 git credential manager 自动认证
- **不重复 clone**，已有本地仓库时直接用

---

## 输入参数

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| source | string | 是 | - | 本地源目录路径（skill 目录或任意目录） |
| repo | string | 是 | - | GitHub 仓库名（不含 owner） |
| owner | string | 否 | 从 git credential 获取 | GitHub 用户名 |
| sub_dir | string | 否 | 仓库根目录 | 源文件放在仓库的子目录路径（如 "examples/故事名"） |
| description | string | 否 | 无 | 仓库描述（仅新建仓库时使用） |
| private | bool | 否 | false | 是否创建私有仓库 |

---

## 执行流程

```
1. 获取 GitHub 认证（git credential fill）
2. 确认仓库是否已存在
   ├── 已存在 → clone 到临时目录
   └── 不存在 → 通过 API 创建，然后 clone
3. 将源目录内容复制到 clone 目录（可选子目录）
4. 排除 __pycache__/、.pyc、.lock 文件
5. git add -A && git commit && git push
6. 报告结果
7. 可选：删除临时 clone 目录
```

---

## 获取认证

```bash
git credential fill <<EOF
protocol=https
host=github.com

EOF
```

输出中 `password=` 后的值即为 token。每次获取后立即使用，不要缓存或持久化。

---

## 创建仓库（仅新仓库）

```python
import requests

resp = requests.post(
    "https://api.github.com/user/repos",
    headers={"Authorization": "token {TOKEN}", "Content-Type": "application/json"},
    json={
        "name": "{REPO}",
        "description": "{DESCRIPTION}",
        "auto_init": True,
        "private": False
    }
)
# 201 = 创建成功
```

---

## Clone + Copy + Push

```bash
# Clone
git clone https://github.com/{OWNER}/{REPO}.git {TEMP_DIR}

# Copy（Python 更可靠，处理中文路径）
python -c "
import shutil, os
src = '{SOURCE}'
dst = os.path.join('{TEMP_DIR}', '{SUB_DIR}')
if os.path.exists(dst):
    shutil.rmtree(dst)
shutil.copytree(src, dst)
"

# 排除不需要的文件
python -c "
import os
root = '{TEMP_DIR}'
for dirpath, dirnames, filenames in os.walk(root):
    dirnames[:] = [d for d in dirnames if d != '__pycache__']
    for f in filenames:
        if f.endswith('.pyc') or f.endswith('.lock'):
            os.remove(os.path.join(dirpath, f))
"

# Commit & Push
cd {TEMP_DIR}
git config user.email "{OWNER}@users.noreply.github.com"
git config user.name "{OWNER}"
git add -A
git commit -m "{COMMIT_MESSAGE}"
git push origin main
```

---

## 安全规则

1. **不硬编码 token**：源代码中不得包含任何 GitHub token 默认值
2. **不缓存 token**：每次从 git credential 获取，用完即弃
3. **排除敏感文件**：自动跳过 `.env`、`credentials.json`、`*.key`、`*.secret`
4. **推送前检查**：扫描待推送文件，发现敏感文件时警告用户并跳过

---

## 输出

```
✅ 推送成功
仓库: https://github.com/{OWNER}/{REPO}
文件数: {N}
最新提交: {COMMIT_SHA}
```

---

## 注意事项

1. 中文路径用 Python shutil.copytree 处理，不用 xcopy/cp（编码问题）
2. git config user.email/name 在 clone 目录内设置（局部），不修改全局配置
3. clone 产生的临时目录可由用户决定是否删除
4. 如果仓库已有内容，新增文件不会覆盖已有文件（git 合而非替换）
5. push 用 HTTPS + credential manager，不需要手动输入密码