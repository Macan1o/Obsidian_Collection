
---

> [!NOTE] 背景
> 核心需求就是当前分支远程已经有了，需要将 commit 封存然后切换到新的分支后再拿出来

```
# 在A分支，把本地commit撤销，改动放回工作区（不丢代码！）
git reset --soft HEAD~1

# 暂存所有改动
git stash

# 切到目标分支B
git checkout -b xxxx

# 把暂存的改动拿出来
git stash pop
```

重新提交即可