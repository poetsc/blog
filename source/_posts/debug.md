---
title: Egg.js debugging in VS Code
date: 2022-01-04 16:54:03
tags:
---

> Visual Studio Code编辑器内置了对Node.js运行时的调试支持，可以调试JavaScript、TypeScript和许多其他编译成JavaScript的语言。
<!--more-->

##  使用 egg-bin 

---

### 添加命令

添加 `npm scripts` 到 `package.json`：

```
{
  "scripts": {
    "debug": "egg-bin debug"
  }
}
```

这样我们就可以通过 `npm run debug` 命令来断点调试应用。

> 开发阶段 worker 在代码修改后会热重启


---

### 环境配置

- 执行 debug 命令时，应用也是以 env: local 启动的，读取的配置是 config.default.js 和 config.local.js 合并的结果
- 启用VSCode `Auto Attach` 功能，有两种方式： 
   - 在命令面板(⇧⌘P)中使用 `Toggle Auto Attach` 命令，切换到`smart（智能）`选项，回车即可
   - 在设置里搜索 `debug javascript: auto attach filter`，选择`smart（智能）`选项并保存

 

---

## 调试

调试启动后，调试工具栏将出现在编辑器顶部，从左到右分别是

- Continue / Pause （继续/暂停）
- Step Over （跳过）  
> 继续执行当前作用域中的下一行(即转到下一行)，而不向下执行任何方法调用。通常用于在特定方法中遵循逻辑，而不用考虑实现细节，对于发现方法中什么地方违反了预期条件非常有用。

- Step Into （步入）  
> 使调试器进入到当前行上的任何方法调用。如果有多个方法，将按执行顺序访问；如果该行没有方法，则效果与跳过相同。即让调试器跟随解释器执行的每一行。

- Step Out （跳出）  
> 一直执行到下一个“返回”或等价的操作，即继续执行直到控制返回到前一个堆栈帧。通常用于当你已经在这个方法里看到了所有你需要的信息，并想要返回到堆栈上层实际使用的地方。

- Restart （重启）
- Stop （停止）

---

## 断点

可以通过单击编辑器边距来切换断点。更精细的断点控制(启用/禁用/重新应用)可以在左侧Run视图的BREAKPOINTS部分中操作。

- 编辑器页边距中的断点通常显示为红色填充圆。
- 禁用的断点有一个填充的灰色圆圈。
- 当调试会话启动时，无法向调试器注册的断点将变为一个灰色的空心圆。
- `Logpoint`是断点的变体，它不会“中断”到调试器，而是将消息记录到控制台。

---

## 变量查看和监视

- 变量可以在Run视图的`Variables`部分中查看，或者将鼠标悬停在编辑器中的源代码上。
- 变量自动归为三类：`Local`、`Closure`和`Global`
- 变量值可以通过变量右键菜单中的`Set Value`进行修改。
- 左侧Run视图的为`WATCH`（监视）面板可以手动输入变量表达式或者执行计算操作。

---

## 更多

更多 Debug 用法可以参考资料：

> [https://eggjs.org/zh-cn/core/development.html#调试](https://eggjs.org/zh-cn/core/development.html#%E8%B0%83%E8%AF%95)
[https://code.visualstudio.com/docs/editor/debugging](https://code.visualstudio.com/docs/editor/debugging)
[https://code.visualstudio.com/docs/nodejs/nodejs-debugging](https://code.visualstudio.com/docs/nodejs/nodejs-debugging)

