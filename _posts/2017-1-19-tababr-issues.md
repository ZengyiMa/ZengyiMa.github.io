---
layout: post
title: "popToRootViewControllerAnimated导致tabbar隐藏的问题"
author: "Damien"
# catalog: true
tags:
    - iOS
    - iOS 问题集
--- 



在 iOS 6 以上使用`UINavigationController`使用`popToRootViewControllerAnimated`并且需要将 tab 切换到某个 index 的时候，如果这么用


```
[self.navigationController popToRootViewControllerAnimated:YES];
[self.tabBarController setSelectedIndex:2];
```

会导致 tabbar 被 hidden ，结果是出现 tabbar 部分出现一片白色。
解决方案是将行代码的顺序对调也就是

```
[self.tabBarController setSelectedIndex:2];
[self.navigationController popToRootViewControllerAnimated:YES];
```

先设置 index ，在来 popToRoot 这样就可以避免这个问题啦




