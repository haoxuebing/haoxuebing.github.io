---
title: git 常用命令
date: 2022-07-03
categories: git
tags: [git]
description: git 分支、版本
---


## git提交代码
**存入缓存区，提交至本地**
```
git add .

git commit -m "message"
```

**直接提交**
```
git commit -am "message"
```

## git 版本切换
**查看版本**
```
git reflog
或
git log
```

**切换版本**
```
git reset --hard 版本号
```

## git 分支

**查看分支**
```
git branch -v
```

**创建分支**
```
git branch 新分支名
```

**删除分支**
```
git branch -d branchname

```

**切换分支**
```
git checkout branchname
```

**创建新分支并切换**
```
git checkout -b 'newbranch'
```

**创建本地分支关联远程分支**
```
git checkout -b [分支名] [远程名]/[分支名]
```

**合并分支**
```
git merge branchname
```

## git 标签管理

**创建标签**
```
git tag -a '标签号' -m '标签信息'
```

**提交标签**
```
git push origin master --follow-tags
```
