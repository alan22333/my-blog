---
title: JSXorJS
description: '小白疑惑：React项目中文件扩展名的选择，JSX还是JS？'
tags: [js]
toc: false
date: 2025-05-04 17:07:58
categories:
---
在React项目中，文件扩展名通常使用 `.js` 或 `.jsx`，选择其中一个主要取决于以下几个因素：

---

### 1. **语法区别**

- `.jsx` 文件通常用于包含 **JSX 语法**（即类似 HTML 的语法，用于描述 UI 结构）。

- `.js` 文件可以包含普通的 JavaScript 代码，通常用于逻辑、数据处理、工具函数等，不一定涉及 JSX 代码。

**示例：**

**`App.jsx`（包含JSX的组件）**

```Shell
function App() {
  return <h1>Hello, React!</h1>;
}

export default App;
```


**`utils.js`（普通JS代码，无JSX）**

```JavaScript
export function add(a, b) {
  return a + b;
}
```


---

### 2. **社区惯例**

在早期，React 社区通常使用 `.jsx` 作为包含 JSX 语法的文件，以便区分哪些文件使用了 JSX。但随着现代 JavaScript 生态（比如 Webpack、Babel 等工具）普遍支持 JSX 解析，许多开发者认为 `.jsx` 变得不那么必要，直接使用 `.js` 也可以处理 JSX 代码。

**实际情况：**

- 有些团队或个人仍然喜欢用 `.jsx` 以明确文件用途。

- 但大部分现代 React 项目（如 Create React App、Next.js）默认使用 `.js`，因为 Babel 已自动处理 JSX 语法，无需显式区分。

---

### 3. **工具链支持**

如果你使用的工具链（如 ESLint、Prettier、Babel）对 `.jsx` 有特殊的 lint 规则或格式化方式，可能更适合使用 `.jsx`。否则，`.js` 足够满足需求。

---

### 4. **团队一致性**

- 在团队协作中，统一使用 `.js` 或 `.jsx` 有助于提高代码一致性，减少认知负担。

- 一些团队更倾向于将逻辑分离：

  - 逻辑处理（无 JSX）使用 `.js`

  - 组件（有 JSX）使用 `.jsx`

---

### 5. **Next.js 的惯例**

Next.js 官方建议直接使用 `.js` 或 `.ts`，因为它的编译工具默认支持 JSX 解析。因此，在 Next.js 项目中，通常不会使用 `.jsx` 扩展名。

---

### **总结：何时使用哪种？**

|文件类型|适用扩展名|推荐实践|
|-|-|-|
|纯 JavaScript 逻辑|`.js`|✅|
|React 组件 (JSX)|`.jsx` 或 `.js`|随项目规范选择|
|TypeScript 文件|`.ts` 或 `.tsx`|✅|

如果你喜欢明确区分组件，使用 `.jsx`。如果你想保持简单，直接用 `.js` 也是完全可以的。

---

如果你是一个人开发或团队没有特别规定，推荐统一使用 `.js` 以减少额外的区分工作。



