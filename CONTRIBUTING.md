# 贡献指南

感谢你对 Full Stack FastAPI Template 做出贡献的兴趣！🙇

## 先讨论

对于**重大变更**（新功能、架构变更、重要的重构），请先在 [GitHub Discussion](https://github.com/fastapi/full-stack-fastapi-template/discussions) 中发起讨论。这样社区和维护者可以在你投入大量时间实现之前，对方案提供反馈。

对于小而直接的变更，你可以直接提交 Pull Request，无需先发起讨论。这包括：

- 拼写和语法修复
- 小型可复现的 bug 修复
- 修复 lint 警告或类型错误
- 轻微的代码改进（例如，删除未使用的代码）

请注意，非团队成员提交的 PR 不允许修改 `pyproject.toml` 或 `uv.lock`，以防止供应链风险。
如果你想添加新的依赖项，请创建一个新的 [Discussion](https://github.com/fastapi/full-stack-fastapi-template/discussions) 来说明原因。

## 开发

有关设置开发环境、运行技术栈、代码检查、pre-commit 钩子等的详细说明，请参阅[开发指南](development.md)。

## Pull Request

提交 Pull Request 时：

1. 确保所有测试在提交前通过。
2. 保持每个 PR 聚焦于单一变更。
3. 如果你改变了现有功能，请更新相应的测试。
4. 在 PR 描述中引用任何相关 issue。

## 自动化代码与 AI

我们鼓励你使用任何你想要的工具来高效地完成工作和贡献，这包括 AI（LLM）工具等。然而，贡献应当包含有意义的人工干预、判断和上下文等。

如果某个 PR 中投入的**人工努力**（例如编写 LLM 提示词）**少于**我们**审查它**所需的**努力**，请**不要**提交该 PR。

可以这样理解：我们自己也可以编写 LLM 提示词或运行自动化工具，而且这样做比审查外部 PR 更快。

### 关闭自动化与 AI 生成的 PR

如果我们看到看起来像是由 AI 生成或以类似方式自动生成的 PR，我们会标记并关闭它们。

这条规则同样适用于评论和描述，请不要复制粘贴 LLM 生成的内容。

### 人力拒绝服务攻击

使用自动化工具和 AI 提交需要我们仔细审查和处理的 PR 或评论，相当于对我们的人力进行[拒绝服务攻击](https://en.wikipedia.org/wiki/Denial-of-service_attack)。

提交 PR 的人只需付出极少的努力（一个 LLM 提示词），却会在我们这边产生大量的工作（仔细审查代码）。

请不要这样做。

对于持续提交自动化 PR 或评论进行骚扰的账户，我们将不得不进行屏蔽。

### 明智地使用工具

正如本叔叔所说：

> 能力越大 ~~能力~~ **工具** 越大，责任越大。

避免无意中造成伤害。

你手中有强大的工具，请明智地使用它们，有效地提供帮助。

## 有问题？

如果你对贡献有任何疑问，欢迎在 [GitHub Discussion](https://github.com/fastapi/full-stack-fastapi-template/discussions) 中提出。
