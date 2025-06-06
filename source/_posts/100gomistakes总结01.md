---
title: 100 go mistakes 总结01
description: '《100 go mistakes》第二章内容总结'
tags: ['go']
toc: false
date: 2025-06-05 09:17:43
categories:
    - go
    - basic
---

**Summary**

* **Avoiding shadowed variables can help prevent mistakes like referencing the wrong variable or confusing readers.**
    * 避免使用遮蔽变量可以帮助防止诸如引用错误变量或混淆读者之类的错误。

* **Avoiding nested levels and keeping the happy path aligned on the left makes building a mental code model easier.**
    * 避免嵌套层级并将“ happy path”（正常流程）代码靠左对齐，可以使构建代码的思维模型更容易。

* **When initializing variables, remember that init functions have limited error handling and make state handling and testing more complex. In most cases, initializations should be handled as specific functions.**
    * 在初始化变量时，请记住 `init` 函数的错误处理能力有限，并且会使状态处理和测试变得更复杂。在大多数情况下，初始化应作为特定函数来处理。

* **Forcing the use of getters and setters isn’t idiomatic in Go. Being pragmatic and finding the right balance between efficiency and blindly following certain idioms should be the way to go.**
    * 在 Go 语言中，强制使用 getter 和 setter 并不符合 Go 语言习惯。务实并在效率和盲目遵循某些惯例之间找到正确的平衡，才是正确的方法。

* **Abstractions should be discovered, not created. To prevent unnecessary complexity, create an interface when you need it and not when you foresee needing it, or if you can at least prove the abstraction to be a valid one.**
    * 抽象应该被发现，而不是被创造。为了防止不必要的复杂性，请在需要时才创建接口，而不是在你预见到需要时，或者至少你可以证明该抽象是有效的。

* **Keeping interfaces on the client side avoids unnecessary abstractions.**
    * 将接口保留在客户端侧可以避免不必要的抽象。

* **To prevent being restricted in terms of flexibility, a function shouldn’t return interfaces but concrete implementations in most cases. Conversely, a function should accept interfaces whenever possible.**
    * 为了防止在灵活性方面受到限制，函数在大多数情况下不应返回接口，而应返回具体实现。相反，函数应尽可能接受接口。

* **Only use `any` if you need to accept or return any possible type, such as `json.Marshal`. Otherwise, `any` doesn’t provide meaningful information and can lead to compile-time issues by allowing a caller to call methods with any data type.**
    * 仅在需要接受或返回任何可能的类型（例如 `json.Marshal`）时才使用 `any`。否则，`any` 不提供有意义的信息，并且可能通过允许调用者使用任何数据类型调用方法而导致编译时问题。

* **Relying on generics and type parameters can prevent writing boilerplate code to factor out elements or behaviors. However, do not use type parameters prematurely, but only when you see a concrete need for them. Otherwise, they introduce unnecessary abstractions and complexity.**
    * 依赖泛型和类型参数可以防止编写样板代码来分解元素或行为。但是，不要过早地使用类型参数，而只在您看到具体需要时才使用它们。否则，它们会引入不必要的抽象和复杂性。

* **Using type embedding can also help avoid boilerplate code; however, ensure that doing so doesn’t lead to visibility issues where some fields should have remained hidden.**
    * 使用类型嵌入也可以帮助避免样板代码；但是，请确保这样做不会导致可见性问题，即某些字段应该保持隐藏。

* **To handle options conveniently and in an API-friendly manner, use the functional options pattern.**
    * 为了方便且以 API 友好的方式处理选项，请使用函数式选项模式。

* **Following a layout such as project-layout can be a good way to start structuring Go projects, especially if you are looking for existing conventions to standardize a new project.**
    * 遵循诸如 project-layout 之类的布局是开始组织 Go 项目的好方法，特别是如果您正在寻找现有约定来标准化新项目。

* **Naming is a critical piece of application design. Creating packages such as `common`, `util`, and `shared` doesn't bring much value for the reader. Refactor such packages into meaningful and specific package names.**
    * 命名是应用程序设计的关键部分。创建诸如 `common`、`util` 和 `shared` 之类的包对读者来说价值不大。将这些包重构为有意义和具体的包名。

* **To avoid naming collisions between variables and packages, leading to confusion or perhaps even bugs, use unique names for each one. If this isn’t feasible, use an import alias to change the qualifier to differentiate the package name from the variable name, or think of a better name.**
    * 为了避免变量和包之间的命名冲突，导致混淆甚至错误，请为每个使用唯一的名称。如果这不可行，请使用导入别名来更改限定符，以区分包名和变量名，或者考虑一个更好的名称。

* **To help clients and maintainers understand your code’s purpose, document exported elements.**
    * 为了帮助客户端和维护者理解代码的目的，请为导出的元素编写文档。

* **To improve code quality and consistency, use linters and formatters.**
    * 为了提高代码质量和一致性，请使用代码检查工具（linters）和格式化工具（formatters）。