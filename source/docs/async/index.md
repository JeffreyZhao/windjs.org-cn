---
layout: page
title: Wind.js异步模块
---

{% render_partial docs/async/_index.md %}

## 引入模块

Wind.js可以在各种JavaScript的执行环境中使用。

### 浏览器

{% render_partial docs/async/importing/_browser.md %}

### Node.js

{% render_partial docs/async/importing/_node.md %}

### 其他环境

Wind.js还支持AMD等包加载环境，更多内容请参考[模块引入相关文档]({{root_url}}/docs/importing/)。

加载Wind.js异步模块之后，便可以定义并使用“[异步方法](./method.html)”了。