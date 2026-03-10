---
title: 编写高效的CLAUDE.md
date: 2026-03-09 17:02:13
tags:
  - claude
  - agent
categories:
  - AI
  - 编程
---

CLAUDE.md是claude code 记忆系统的一部分， 用于claude了解你的项目背景，明确它在干活时应该遵循的一系列底层规则。如：项目架构、使用技术栈、编码标准、常用工作流等。

CLAUDE.md写得好不好，直接决定了claude是靠谱的同事，还是每次都要重新培训的实习生。所以CLAUDE.md也像一份入职手册， 让claude通过这份入职手册成为你的靠谱的同事。

## 编写CLAUDE.md的核心原则

为了创建一个好的 CLAUDE.md，你需要遵循一些核心原则。 如下

### 核心原则1: Less is more

 CLAUDE.md的每一行都会流入到每一次对话的上下文中；冗余的内容会持续消耗tokens。 所以尽量让CLAUDE.md只包含必要的内容，避免重复。

### 核心原则2: 具体优于泛泛

下面以一个非常常见、但几乎没有任何效果的写法：

```md
# 项目规范
## 代码质量
请写出高质量的代码。代码应该是可读的。使用有意义的变量名。
保持代码整洁。遵循最佳实践。不要写重复的代码。
```

这些话没有错， 但问题在于claude本来就知道这些，它不会改变claude的任何决策，只会白白占用上下文空间。 所以， 更好的写法是：

```md
# 项目规范
## java
- 遵循google java style guide
- 使用lombok
- 函数参数 > 3 个时需使用对象
## 错误处理
- 业务错误使用 throw new BusinessException(code, message)
- controller 层不要使用try catch，由全局异常处理类处理
- service 层不要直接返回View层对象， 而是由controller层处理
```

两者的差异在于， 后者不是模糊的要求“高质量的代码”， 而是给出了如何做才算高质量的代码。

这里有一个简单的判断标准：如果你不写， claude也大概率会做对，那就不要写。

### 核心原则3: 关键三问题 WHY, WHAT, HOW

CLAUDE.md中，通常都在回答这三类问题。为claude给出明确的指引。

#### WHY -- 为什么要这样做？

作用：探寻原因、动机或目的。让claude理解行为动机，理解决策逻辑。
当claude明白*为什么*，它在面对相似但不完全相同的场景时，才更能做出一致的判断。

```md
## 在 MyBatis 开发中，为什么选择 XML 映射文件而不是注解(Annotations)？
- XML 支持原生换行和缩进，SQL 结构清晰，便于阅读和调试
- XML 原生支持所有动态 SQL 标签，逻辑清晰，编写和修改都非常方便
- 简单SQL使用annotation 映射，比 XML 更加简洁, 但是MyBatis-Plus 等增强框架，大部分基础 SQL 已自动生成

```

#### WHAT -- 具体做什么，不做什么？

作用：获取具体信息、定义或事实,明确对象、澄清概念。这一部分的重点是边界。什么是允许的，什么是禁止的，决策应该发生在哪一层？

```md
## 数据库操作规范
- 所有查询通过 Mybatis-plus 进行
- 禁止在 controller/service 中直接写 SQL
```

#### HOW -- 怎么做？什么步骤？

作用：了解方法、过程或机制。当步骤清晰、路径明确、还有参考文件时，Claude 才会稳定复用同一套工作流，而不是每次自由发挥。

```md
## 创建新 API 端点
1. 在 `src/main/java/**/vo/` 创建请求/响应的schema
2. 在 `src/main/java/**/controller/` 实现请求处理
3. 在 `src/main/java/**/service/` 实现业务逻辑
4. 在 `src/test/java/**/` 添加测试用例

示例参考: `src/controller/OrderController.java`
```

### 核心原则 4：渐进式披露：不要把一切都塞进 CLAUDE.md

CLAUDE.md 的职责是定义默认决策，而不是承载全部知识。对于非核心、但可能被用到的内容，正确的做法是引用，而不是复制。

```md
# 项目规范
## 核心
[精简规范]

## 详细文档
- 数据库设计: 见 `docs/database.md`
- API 规范: 见 `docs/api-spec.md`
- 部署流程: 见 `docs/deployment.md`
```

## 如何快速创建一个 CLAUDE.md

如果你已经有一个项目，并且大多数据项目的技术栈、编码规范都是固定的，那么你可以通过`/init` 命令快速创建一个`CLAUDE.md`。

Claude 会根据你的项目现状生成一个适合的`CLAUDE.md`，在此过程中可以找一个比较小的项目，来进行创建因为claude会尽可能的去获取项目信息，但是项目越大，claude获取的信息就越多， 这个过程可能会消耗比较多的tokens。通过这个命令，claude会将项目的技术栈、编码规范、目录结构、常用命令等信息，生成一个 CLAUDE.md，并保存在项目根目录下。

后面你可以手动修改这个`CLAUDE.md`，添加更多的内容。

这个`CLAUDE.md`可以作为你的项目模板，复用到其它项目中。
