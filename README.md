## WeeklyBlog
**欢迎来到 itsCoder 的 WeeklyBlog 项目。**
### 项目简介
本项目旨在督促成员利用业余时间继续学习，以写博客的方式或者开源项目作为产出。在提升自己的同时，将技术知识分享到开源社区。

### 加入条件
- 熟悉 Git 操作
- 拥有个人博客
- 认真
- 热爱技术，愿意花时间提升自己
- 无理由连续两期无产出成员，将从项目中移除

### 开展方式
- 时间：每两周为一周期
- 目标：每期每个成员至少有一篇博文或者开源项目产出
- 审阅：每期结束之前，产出由项目内成员轮转审阅，参照产出标准，给出审阅意见，可以加入评分机制
- 发布：产出由成员各自发布到自己的博客或者其他平台，每期优秀的产出放在 [ itsCoder 主页](http://www.itscoder.com) 发布

### 产出标准
- 形式不限，文章、开源项目均可
- 领域不限，Android、iOS、Python 等均可
- 内容等级不限，入门或者深入的都可以，根据成员自身情况自己决定
- 错别字、语句错误尽可能避免
- 文章排版参考 [文案风格指南](https://open.leancloud.cn/copywriting-style-guide.html)
- 最终的标准：认真对待

### 项目规范
- 分支说明
  - master: 主分支
  - dev: 每期进行中分支，每期结束后合并到 master 分支
  - member/member_id: 成员分支，fork 回自己的仓库后建立的成员分支

- 标签说明

  - **申请加入**：申请加入本项目
  - **第 x 期**：每一期的标签
  - **认领审阅**：每期认领审阅文章的标签
  - **审阅完成**：完成本期审阅
  - **完成**：根据审阅意见修改完毕，完成本期的作业

- Pull requests 规范
  - 每个 pr 的标题明确本次 pr 主要内容

  - 申请加入的 pr 提交到 master 分支

  - 除了申请加入 pr 外，其余 pr 均提到 dev 分支

  - 申请加入标题为 `xxx申请加入` ，每期的文章提交标题为 `第 x 期：文章标题 by 你的id` ，例如：

    ```
    第 2 期：框架源码 -- Retrofit 简析学习 by hugo
    ```

    期数使用阿拉伯数字，前后空一格。

- 加入规范：

  1. 非 itsCoder 成员可以在本项目下提 issue，提交申请信息，包括博客地址、个人介绍等，项目负责人审核通过后加入，即可参与本项目

  2. 申请通过后，fork 本项目到自己的仓库，新建一个成员分支，分支名：`member/your_id`，分支名全部小写，例如：`member/jaeger`

  3. 然后在 README.md 的项目成员块中添加自己的信息，格式如下，确保 id 和上一步中使用的 id 相同，大小写可以不一致：

     ```
     昵称 [@id]( GitHub 主页)
     示例：写代码的猴子 [@Jaeger](https://github.com/laobie)
     ```

  4. 将本地分支 push 到自己仓库对应的远程分支，并向本项目的 `master` 分支提交 pr，标题“xxx申请加入”

- 开发规范：

  1. fork 到自己的仓库

  2. 切到 dev 分支
  3. 从 dev 分支 切一个新的 member 分支
  4. 在自己的 member 分支添加、改动、提交，每个成员只允许在自己成员分支上进行操作

  5. 每期项目新建一个文件夹，文件夹命名规则为：`phase_期数`，例如：`phase_3` 即为第三期

  6. 文章均使用 Markdowm 格式，命名为`id_日期(yyyyMMdd)_title.md`(全部小写，下划线连接),例如：`jaeger_20160606_how_to_use_vector_drawable.md` 

  7. 文章完成后 push 到自己仓库成员对应的远程分支，并提 pr 至 dev

  8. 基于每次 pr 进行审阅，提出修改或者有疑问的评论，审阅完毕给出评价，并标上 **审阅完成** 标签

  9. 作者根据审核意见进行修改，修改完成后，标记上 **完成** 标签，并在 pr 的评论里给出自己博客上的地址，格式如下：

     ```
     [文章标题](文章链接) ([@作者名](作者主页地址，可以为 GitHub 地址))
     ```

  10. 完成之后由负责人合并到 dev 分支，每期结束时 dev 分支合并到 master 分支

- 审阅规范：

  1. 审阅时间为下一期的第一周，时间宽松，需要保证审阅质量
  2. 审阅人员优先认领，认领事标记上**认领审阅**标签，后续再通过其他方式指定，每篇文章保证有人审阅
  3. 基于 pr 进行审阅，在需要修改或者不理解的地方添加评论
  4. 审阅完成后，从叙述方式、格式规范、改进建议等角度给出审阅评价

- 发布规范

  1. 文章正文前建议注明本项目的相关信息（不做硬性规定），内容和格式如下

     ```
     >- 文章来源：itsCoder 的 [WeeklyBolg](https://github.com/itsCoder/weeklyblog) 项目
     >- itsCoder主页：[http://itscoder.com/](http://itscoder.com/)
     >- 作者：[]()
     >- 审阅者：[]()
     ```

  2. 文章完成审阅之后发布到个人博客 ，并将个人博客上的文章地址更新到 pr 对话中，参照开发规范第 5 条

  3. 每期完成后由负责人发布到 [itsCoder 主页](http://itscoder.com/)

#### 项目成员
- 写代码的猴子 [@Jaeger](https://github.com/laobie)

- Hymanme [@hymane](https://github.com/hymanme)

- Brucezz [@Brucezz](https://github.com/brucezz)

- Melodyxxx [@Melodyxxx](https://github.com/melodyxxx)

- allenwu [@allenwu](https://github.com/wuchangfeng)

- 黑丫山上小旋风 [@ExtremeJoe](https://github.com/JoeSteven)

- 谢三弟 [@IMXIE](https://github.com/xcc3641)

- 用语 [@yongyu](https://github.com/yongyu0102)

- 写代码的香港记者 [@Manjusaka](https://github.com/Zheaoli)

- Melo [@Melo](https://github.com/itsMelo)

- shadow [@shaDowZwy](https://github.com/shaDowZwy)

- Win-Man [@Win-Man](https://github.com/Win-Man)

- JangGwa [@JangGwa](https://github.com/JangGwa)

- showzeng [@showzeng](https://github.com/showzeng)
