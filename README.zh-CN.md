<div align="center">

# go 中文文档

**[中文版] go — Google 开源的 Go 编程语言官方仓库**

[![原项目](https://img.shields.io/badge/原项目-golang--go-blue?style=flat-square&logo=github)](https://github.com/golang/go)
[![微信联系](https://img.shields.io/badge/微信-uaycar-brightgreen?style=flat-square&logo=wechat)](#)

</div>

---

> 本文是 [golang/go](https://github.com/golang/go) 项目 README 的中文翻译版本。
> 完整源代码请访问原项目:https://github.com/golang/go

---

# Go 编程语言

Go 是一门开源编程语言,让构建简单、可靠、高效的软件变得容易。

![Gopher 图片](https://golang.org/doc/gopher/fiveyears.jpg)
*Gopher 图片由 [Renee French][rf] 创作,基于 [Creative Commons 4.0 署名许可证][cc4-by] 授权使用。*

Go 的官方规范 Git 仓库位于 https://go.googlesource.com/go ,
https://github.com/golang/go 是该仓库的镜像。

除非另有说明,Go 的源代码文件均依据 LICENSE 文件中的 BSD 风格许可证发行。

## 仓库说明

本仓库(Go 语言源码仓库)承载了 Go 编译器、标准库、运行时、工具链以及相关文档的完整源代码。日常开发者通常不需要直接克隆本仓库,而是通过官方二进制发行版安装;只有在阅读源码、参与贡献或需要自行编译时才需要接触本仓库内容。

### 下载与安装

#### 二进制发行版

官方二进制发行版可从 https://go.dev/dl/ 获取,覆盖 Windows、macOS、Linux 等主流操作系统及常见 CPU 架构,这是绝大多数用户推荐的安装方式。

下载二进制发行版后,请访问 https://go.dev/doc/install
查看安装说明。

#### 从源码安装

如果你的操作系统与架构组合没有对应的二进制发行版,或出于定制、研究目的需要自行编译,请访问
https://go.dev/doc/install/source
查看从源码安装的说明。源码安装方式通常需要先准备一份较新的 Go 工具链或引导工具链,具体要求以官方文档为准。

### 安装方式速查

| 安装方式 | 适用场景 | 参考链接 |
|:---------|:---------|:---------|
| 官方二进制发行版 | 日常开发,推荐首选 | https://go.dev/dl/ |
| 二进制版安装说明 | 下载后配置环境 | https://go.dev/doc/install |
| 从源码编译安装 | 无对应发行包或需要定制 | https://go.dev/doc/install/source |

### 参与贡献

Go 是数千名贡献者共同努力的成果,我们感谢你的帮助!

如需参与贡献,请先阅读贡献指南:https://go.dev/doc/contribute 。贡献流程、代码评审要求与提交规范均以该指南为准。

请注意,Go 项目的问题跟踪器(issue tracker)仅用于缺陷报告和提案。
关于 Go 语言的使用问题,请参考 https://go.dev/wiki/Questions
中列出的提问渠道,例如社区论坛、邮件列表与聊天频道等。

### 常见问题

- **问:应该从哪里下载 Go?**
  答:官方下载页面为 https://go.dev/dl/ ,请勿使用非官方来源的安装包。

- **问:我的平台没有官方二进制包怎么办?**
  答:参考 https://go.dev/doc/install/source 从源码自行编译安装。

- **问:遇到 Go 语言使用问题应该在哪里提问?**
  答:原仓库的 issue 仅用于缺陷报告和提案,语言使用问题请走 https://go.dev/wiki/Questions 列出的社区渠道。

- **问:本项目采用什么许可证?**
  答:除非另有说明,Go 源代码文件均依据 LICENSE 文件中的 BSD 风格许可证发行。

## 学习资源

- 官方文档首页:https://go.dev/doc/
- 交互式入门教程 A Tour of Go:https://go.dev/tour/
- 标准库文档:https://pkg.go.dev/std
- 贡献指南:https://go.dev/doc/contribute

[rf]: https://reneefrench.blogspot.com/
[cc4-by]: https://creativecommons.org/licenses/by/4.0/

---

## 版权与声明

- 本文档为 [golang/go](https://github.com/golang/go) 项目 README 的中文翻译版本,仅供中文开发者学习参考,翻译内容与原 README 不一致处以原文为准。
- Go 项目源代码版权归原项目作者所有,遵循其原始的 BSD 风格许可证(详见原仓库 LICENSE 文件)。
- 本仓库不包含任何 Go 源代码副本,原项目与最新代码请以 https://github.com/golang/go 为准。

**代部署 / 定制服务 / 技术咨询 请添加微信:uaycar**

**如果觉得有用,请给原项目点个 Star!** ⭐
