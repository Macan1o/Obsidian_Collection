---
created: 2026-09-11T14:28
---


---
### 首先跟远程仓库建立联系

```
git remote add origin ssh://git@gitlab.shinet.top:20755/common/group_uniapp/uniapp.git
```

### 建立本地分支并切换到该分支

```
git checkout -b gX/XXXXXXXXX
```

### 推送到远程仓库对应分支

```
git push -u origin gX/XXXXXXXX
```

### 如果需要强行覆盖远程分支「慎用」

```
git push -f origin
```