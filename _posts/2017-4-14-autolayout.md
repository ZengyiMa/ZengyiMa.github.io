---
layout: post
title: "关于 Autolayout"
author: "Damien"
# catalog: true
tags:
    - iOS
    - Objective-C
--- 

 
Autolayout 是基于 Cassowary 算法实现的一套 UI 布局算法，能够有效解析线性等式系统和线性不等式系统，用户的界面中总是会出现不等关系和相等关系，Cassowary开发了一种规则系统可以通过约束来描述视图间关系。约束就是规则，能够表示出一个视图相对于另一个视图的位置。

# Autolayout 布局流程
1. 更新约束，计算布局
当我们写好了约束代码之后，如使用系统自带的语法，VFL，Masonary，这组成了一个页面的约束条件，在一个 View 显示到屏幕上面之时，首先会发现更新约束，你也可以调用 setNeedsUpdateConstraints 来手动出发，通常系统会自动触发此方法，随后系统会在需要的时候调用 updateConstraintsIfNeeded 方法来表示立即更新 约束，你也有重写 updateConstraints 来修改约束。

2. 布局视图
此方法如 layoutSubviews 会将计划好的布局的 frame 一个个设置到子视图中。你可以通过调用 setNeedsLayout 来触发一个操作请求，这并不会立刻应用布局，而是在稍后再进行处理。因为所有的布局请求将会被合并到一个布局操作中去，所以你不需要为经常调用这个方法而担心。

# Autolayout 其他属性

## Priority 优先级
Priority 在一些场景下还是很有用的，2 个冲突的约束，优先级高的会优先被满足。如，一个既设置了宽度又设置了距离边距的 view，可以适当的调整优先级在屏幕小的情况下来满足最佳的视觉范围。

## intrinsic size （固有尺寸）
我们在将一个 UILabel 或者 UIButton 的添加约束的时候会发现它们并不需要一个宽高也可以完成工作，这就是 intrinsic size 这个在起作用的，它表示这个 View 可以自己获取到合适的大小。

## Content Compression Priority（抗压缩的优先级）
如果一个父视图的大小不过，会优先压缩子视图大小来满足布局，但是有的时候往往只想压缩一下无关紧要的控件大小，这个时候 Content Compression Resistance Priority 就上场了，如果它的优先级越高，那么它越不容易被外界压缩大小。

## Content Hugging Priority（抗拉伸的优先级）
和 Content Compression 相反，Content Hugging 指定的是一个视图被拉伸的优先级，如果值越大，那么越难被拉伸。

