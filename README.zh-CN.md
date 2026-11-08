# Student Resource Site：一个 Claude skill

[English](README.md) · **中文**

这是一个 [Claude](https://claude.com) skill，教 Claude 设计和搭建**学生自办的课程资料站加社区**：

- **资料**：往年试卷、课程大纲、作业说明、实验指导书和笔记，按课程和学期整理。
- **功能**：上传通道、课程评分、匿名讨论区、机会墙、博客和问答小帮手。
- **成本**：固定成本为零，AI 花费按美元封顶。

skill 的内容来自从头搭建并运营这样一个网站的经验。里面每条要求都写了原因，很多原因就是当初真出过的问题，后来定成了规则。

## 它让 Claude 学会什么

请 Claude 规划、搭建、扩展或检查这类网站时，哪怕只问其中一块（比如"上传页面该怎么做？""怎么防止有人上传带学号的文件？"），这个 skill 会给它：

- **三类用户的要求**：考试周拿手机找资料的读者，没耐心的上传者，只有一两个人、时间很少的志愿维护者。
- **9 条设计原则**，其中包括：
  - 隐私不确定就交给人；
  - 从不悄悄丢掉别人上传的文件；
  - 仓库是唯一的内容来源；
  - 只评课，不评人；
  - 用修课学期而不是"大二"来标时间；
  - 每次付费 AI 调用都有美元上限。
- **分阶段的做法**：从开头要问清的问题一直到上线，每个阶段都写明怎样才算做完。
- **详细的要求清单**，每条附原因：

| 部分 | 参考文件 |
|---|---|
| 目录结构、文件命名、学期、课程表格、哪些内容能收 | [`content-model.md`](plugins/student-resource-site/skills/student-resource-site/references/content-model.md) |
| 网站：导航、每类页面、搜索、手机端、深浅色、文案 | [`site-ui.md`](plugins/student-resource-site/skills/student-resource-site/references/site-ui.md) |
| 上传表单、上传帮助页、机器人每一步、审核 PR、维护者指令、存量资料导入 | [`upload-flow.md`](plugins/student-resource-site/skills/student-resource-site/references/upload-flow.md) |
| AI 分拣和检查、PDF 个人信息打码、打码后复查、查重、安全评审得到的教训 | [`ai-checks-redaction.md`](plugins/student-resource-site/skills/student-resource-site/references/ai-checks-redaction.md) |
| 课程评分、老师名录、讨论区、机会墙、博客、小帮手、内容审核 | [`community.md`](plugins/student-resource-site/skills/student-resource-site/references/community.md) |
| AI 预算和模型选择、免费托管、密钥、总开关、测试、上线、出事预案 | [`operations.md`](plugins/student-resource-site/skills/student-resource-site/references/operations.md) |

**默认技术栈：**
- GitHub 仓库，用 Actions 构建、Pages 托管；
- Issue 表单加机器人处理上传；
- Cloudflare Worker 和 Durable Objects 跑社区功能；
- Claude API 负责分拣、隐私检查和审核。

换别的平台，这些原则同样适用。

## 安装

### 作为 Claude Code 插件（推荐）

在 Claude Code 里：

```
/plugin marketplace add LucasXQY/student-resource-site-skill
/plugin install student-resource-site@lucasxqy-skills
```

或者在终端里：

```bash
claude plugin marketplace add LucasXQY/student-resource-site-skill
claude plugin install student-resource-site@lucasxqy-skills
```

请求合适时 Claude 会自动用上这个 skill；也可以输入 `/student-resource-site:student-resource-site` 直接调用。以后更新用 `claude plugin marketplace update lucasxqy-skills`。

### 手动安装

把 skill 文件夹复制到个人 skill 目录：

```bash
git clone https://github.com/LucasXQY/student-resource-site-skill.git
cp -r student-resource-site-skill/plugins/student-resource-site/skills/student-resource-site ~/.claude/skills/
```

### 在 claude.ai 上用

从[最新版本](https://github.com/LucasXQY/student-resource-site-skill/releases/latest)下载 `student-resource-site.skill`，在 设置 → 功能 → Skills 里上传。需要你的套餐或组织允许自定义 skill。

## 试一试

装好后可以这样问 Claude：

- "我们是计算机学院的学生会，想做一个网站，让同学共享往年试卷、大纲和笔记，还能匿名讨论课程。只有我们两个人维护，没有经费。请出一份完整设计方案。"
- "同学发给我的试卷 PDF 封面上有学号。请设计一个上传页面，再设计一个机器人，在任何东西公开前先把上传的文件处理好。"
- "想给静态的资料站加课程评分和讨论区，怎么设计才不会被老师投诉、被刷分、被发广告？"

## 有用吗？

用上面三个问题做了测试：每个问题让 Claude 带 skill 和不带 skill 各做一遍，再用 22 条检查给生成的设计文档打分。检查的都是这类网站容易做错的地方，比如：

- 上传表单是否只要求文件；
- 是否提醒附件在审核前就是公开的；
- 打码是真删除内容，还是只盖个黑框；
- 评分是否只打分，并且要达到最少人数才显示；
- 是否没有按 IP 限票（校园网大家共用一个 IP）；
- AI 花费是否按金额封顶。

| | 带 skill | 不带 |
|---|---|---|
| 通过检查 | 20 / 22（92%） | 12 / 22（54%） |

不带 skill 的方案有这些问题：

- 上传表单必填课程代码和学期；
- 收答案；
- 讨论区先发后审；
- 允许写文字评价老师；
- 按 IP 限票；
- 要求学校邮箱登录。

带 skill 的方案都避开了。测试用例在 [`evals/`](plugins/student-resource-site/evals)。样本很小，数字只当参考，不算正式评测。

## 仓库结构

```
.claude-plugin/marketplace.json          the plugin marketplace (this repo)
plugins/student-resource-site/
  .claude-plugin/plugin.json             the plugin manifest
  skills/student-resource-site/
    SKILL.md                             principles, phases, which reference to read when
    references/                          the detailed requirement checklists
  evals/                                 test cases (prompts and checks)
```

## 参与贡献

欢迎提 issue 和 PR，特别是这三类：
- 别的学校网站上证明很重要的要求；
- 你踩过、应该定成规则的坑；
- 换到其他平台时的改法。

请守两条规矩：
- 每条要求都写原因。
- 例子里绝不放真实的个人信息，名字和学号一律用一看就是假的。

## 许可证

[MIT](LICENSE)
