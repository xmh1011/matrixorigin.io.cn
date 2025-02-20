# **审阅与评论**

对 MatrixOne 来说，对 PR 的审阅和评论是至关重要的：您可以对他人的 PR 进行分类，以便有专家更快的来解决这些问题；您也可以对代码的内容进行审阅，对代码编写的风格、规范等提出建议；哪怕是在评论区留下一个小小的 Idea，这也弥足珍贵。
所有别再犹豫，别再担心您的想法不够完善，无论多么微小的建议都可能会对 MatrixOne 产生深远影响。

当然，在此之前，我们希望您能认真阅读本节，了解基本要求和相关方法。

## **基本原则**

当对一个 PR 进行评论或者审阅时，无论内容如何，我们呼吁所有参与者都保持友好和善的态度，营造一个和谐的社区氛围。

* **尊重他人**  
尊重每一个 PR 请求的发起人和其他审阅者。代码审阅是社区活动的重要部分，因此请遵循社区要求。

* **注意语气**
在与他人沟通交流时，我们鼓励您多使用 “建议” 或者 “提问” 的语气，而不要总是命令他人。换位思考，所有人都希望被温柔以待！  

* **赞美**  
不要吝啬对他人的赞美！一个好的想法或者好的成果值得我们夸赞。在很多情况下，鼓励、赞美他人往往比不留情面的批评更有价值！  

此外，在具体内容上我们有如下建议：  

* **详细而具体**  
我们希望您能在评论中提供更为详细而具体的信息，因为信息越详细，他人就更容易理解，也可以更高效地解决问题。如果您进行了测试，完全可以放上测试环境和结果；如果您提出了一些建议，那不妨说说应该如何落实。

* **客观而公正**  
请避免个人偏见和主观情绪。诚然，每个人的评论或多或少都会带有主观色彩，但是，作为一个成熟的审阅者，您应该注重技术和数据，而不是个人的喜好。

* **灵活而审慎**  
当遇见一些复杂问题时，即使是在综合考量多方因素之后也很难抉择，“接受” 还是 “拒绝”，是个两难的问题———我们也无法给出一个明确、具体的标准，只能说 “具体情况，具体分析”。但是我们建议您在难以抉择时，寻求他人的帮助。

## **对 PR 分类**

有些 PR 创建者可能并不熟悉 MatrixOne 或相关开发工作流程，因此不确定应该添加何种标签，也不了解应该把问题分配给谁。如果您知道该如何做，我们希望您可以向他们施以援手，为问题补充上标签等信息，这有助于推动问题的解决进程。

## **检查正误**

当您检查代码或其他修改的正误时，务必注意以下几点：

* **聚焦**  
  无论处理的问题有多小，都应该聚焦，因此需要注意的是这项 PR 是否有专注于解决一件事情，是否混淆了其他模块的内容。
* **测试**  
  代码贡献应该确保代码通过测试（单元测试、集成测试等），并提交一份测试报告，展示所使用的测试用例以及代码覆盖率。
* **完善性**  
  审阅代码的完善性主要是关注其是否真的解决了当时提出的问题，与最初的目的是否契合，您可以查看 [GitHub Issue](https://github.com/matrixorigin/matrixone/issues/new/choose) 来追溯整个问题的前因后果与开发走向。
* **风格规范**  
  PR 中的代码应该遵相应的[规范](contribute-code.md#get-familiar-with-style)。现有的部分代码可能与风格指南的要求不一致，您应该维护其一致性，或直接提交一个新的 Issue 来更正它。
* **必要的文档**  
  如果一个 PR 改变了用户构建、测试、交互或发布代码的方式，您必须检查它是否也更新了相关文档，如 `README.md`。类似地，如果一个 PR 删除或弃用了一段代码，您必须检查相应的文档是否也应该被删除。
* **性能**  
  如果您发现新增的代码可能会影响 MatrixOne 的性能，您可以要求开发者提供一个基准测试（如文档中展示的 SSB、TPCH 测试）。
