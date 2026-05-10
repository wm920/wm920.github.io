---
title: "第一篇文章：博客搭建记录"
date: 2026-05-10
draft: false
tags: ["Hugo", "博客"]
categories: ["学习笔记"]
description: "使用 Hugo + PaperMod + GitHub Pages 搭建个人博客的完整过程。"
---

## 为什么搭建博客

记录技术学习笔记，方便自己回顾，也希望能帮助到他人。

## 技术栈

- **静态站点生成器**：[Hugo](https://gohugo.io/)
- **主题**：[PaperMod](https://github.com/adityatelange/hugo-PaperMod)
- **托管**：GitHub Pages（免费）

## 代码高亮示例

```python
def fibonacci(n: int) -> int:
    """返回第 n 个斐波那契数"""
    if n <= 1:
        return n
    a, b = 0, 1
    for _ in range(2, n + 1):
        a, b = b, a + b
    return b

print(fibonacci(10))  # 55
```

## 数学公式示例

行内公式：欧拉公式 $e^{i\pi} + 1 = 0$

块级公式：

$$
\sum_{k=1}^{n} k = \frac{n(n+1)}{2}
$$

## 写文章方法

在 `content/posts/` 目录下新建 `.md` 文件，推送到 GitHub 后自动构建部署。
