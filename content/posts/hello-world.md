---
title: "Hello World — 博客搭建记录"
date: 2026-08-03
draft: false
tags: ["Hugo", "博客", "GitHub Pages"]
categories: ["技术实践"]
slug: "hello-world"
summary: "用 Hugo + GitHub Pages 搭建免费技术博客的完整过程。"
---

## 为什么要搭博客？

学习 AI 的过程中积累了很多笔记和实践经验，需要一个地方系统整理和分享。对比了几个方案后选择了 **Hugo + GitHub Pages**：

- ✅ **完全免费**（GitHub Pages 托管 + 自定义域名可选）
- ✅ **Markdown 写作**（专注内容，不折腾排版）
- ✅ **代码高亮**（适合技术博客）
- ✅ **速度快**（静态站点，CDN 加速）
- ✅ **版本管理**（Git 管理，不怕丢）

## 技术栈

| 组件 | 选择 | 说明 |
|------|------|------|
| 静态生成器 | Hugo | Go 写的，速度极快 |
| 主题 | PaperMod | 极简风格，适合技术博客 |
| 托管 | GitHub Pages | 免费，自动部署 |
| 域名 | coo-moon.github.io | GitHub 免费域名 |

## 一个代码高亮示例

```python
def fibonacci(n):
    """生成斐波那契数列"""
    a, b = 0, 1
    result = []
    for _ in range(n):
        result.append(a)
        a, b = b, a + b
    return result

print(fibonacci(10))
# [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

```javascript
// 一个简单的 debounce 函数
function debounce(fn, delay) {
  let timer = null;
  return function (...args) {
    clearTimeout(timer);
    timer = setTimeout(() => fn.apply(this, args), delay);
  };
}
```

## 下一步计划

- [ ] 写 AI 学习笔记系列
- [ ] 添加评论系统（Giscus）
- [ ] 绑定自定义域名
- [ ] 添加访问统计

---

如果你也想搭建类似的博客，欢迎交流！
