---
layout: post
title: "GitHub fork 的项目代码如何与主项目同步"
author: "Damien"
header-img: "img/github_img.png"
# catalog: true
tags:
    - GitHub
    - Tips
---

在 GitHub 中我们 fork 主项目之后会在我们自己的账号下创建一个 fork 工程，这时候的代码状态就会和主项目断开，在我们需要同步主项目最新状态的时候应该怎么解决这个问题呢？

1、首先拉取主项目 master 分支的最新代码

```
git fetch upstream
```

2、切换到 fork 项目的 master 分支（如果没在）
```
git checkout master
```

3、合并主项目的改动到 fork 项目
```
git merge upstream/master
```

4、push 到远端
```
git push origin master
```

好了，以上就是全部步骤了

官方链接：

https://help.github.com/en/articles/configuring-a-remote-for-a-fork

https://help.github.com/en/articles/syncing-a-fork
