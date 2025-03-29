---
permalink: /
---

## 背景

如果你有使用 vuepress、vitepress、astro 这种文档网站来作为博客然后通过一些云服务厂商的 cos 服务进行部署的需求，那你会发现每次更新完内容后就会需要手动去打包、然后把打包结果放到 cos 服务对应目录下面，如果你有了解 CI/CD，你肯定会想要通过自动化部署来解决问题，还能避免手动误操作导致问题，本着这个想法，我打算尝试一下

## 知识点

在开发之前，让我们先了解一下 [github action](https://docs.github.com/en/actions/about-github-actions/understanding-github-actions) ，它是一个持续集成和持续交付（`CI/CD`）平台，它的工作原理如下：

1. 基于事件的触发系统

- GitHub Actions 通过仓库中的事件来触发工作流程，例如：
- 代码推送（push）
- 拉取请求（pull request）创建
- 问题（issue）创建
- 预定的时间（scheduled）
- 手动触发（workflow_dispatch）

2. 运行环境（Runner）

GitHub Actions 是在服务器上运行的。具体来说：

- GitHub 提供了名为"Runner"的虚拟机环境来执行工作流
- 官方提供了 Linux（Ubuntu）、Windows 和 macOS 虚拟机
- 每个 job 都在一个全新的虚拟机实例上独立运行
- 也可以选择使用自己的服务器作为"自托管运行器"（self-hosted runner）

3. 工作流组件

GitHub Actions 工作流由以下几个主要部分组成：

- 工作流（Workflow）：整个自动化过程的配置
- 作业（Job）：工作流中的独立单元，可以并行或顺序执行
- 步骤（Step）：作业中的最小执行单位
- 动作（Action）：可重用的自动化单元（如 actions/checkout@v3）

4. 执行流程

根据 yaml 配置进行执行我们的流程，下面会讲解

5. 安全凭证管理

GitHub 提供了"Secrets"功能来安全地存储敏感信息（如 API 密钥），可以在工作流中使用如下语法引用。

```yml
${{ secrets.SECRET_NAME }}
```

## 开发

先梳理一下需求：

我目前希望的流程是当我提交代码更新后，通过 github action 来实现监听文件变化，自动打包、同步到腾讯云 cos 服务

### 创建静态网站的存储桶

创建，第一步填写好之后，我们下一步，使用默认即可

![](https://blog-1304565468.cos.ap-shanghai.myqcloud.com/work/1743174063289-6e1cc234-5300-486d-b169-71d35704baba.png)

配置静态网站

![](https://blog-1304565468.cos.ap-shanghai.myqcloud.com/work/1743174285435-dc6924f6-d6ed-463a-b8e6-2d579d262fa2.png)

配置源站：

![](https://blog-1304565468.cos.ap-shanghai.myqcloud.com/work/1743244364851-b4e2290c-b9f5-42c5-a78c-bd74c4c76421.png)

去域名解析添加解析

完成上面的内容后，我们就可以把我们网站内容上传上来了！

### 通过脚本 + github action 实现文件自动上传

```yaml
name: 构建并部署到腾讯云 COS

on:
  push:
    branches: [main] # 可以改为你的主分支名称
  workflow_dispatch: # 允许手动触发

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: 检出代码
        uses: actions/checkout@v3

      - name: 设置 Node.js 环境
        uses: actions/setup-node@v3
        with:
          node-version: "16" # 根据你的项目需求选择版本

      - name: 安装依赖
        run: npm install

      - name: 缓存依赖
        uses: actions/cache@v3
        with:
          path: ~/.npm
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-node-

      - name: 构建
        run: npm run docs:build

      # 添加这个步骤以设置兼容的 Python 版本
      - name: 设置 Python 3.10
        uses: actions/setup-python@v4
        with:
          python-version: "3.10"

      - name: 安装腾讯云 CLI
        run: pip install coscmd

      - name: 配置腾讯云 COS 认证
        run: |
          coscmd config -a ${{ secrets.TENCENT_SECRET_ID }} -s ${{ secrets.TENCENT_SECRET_KEY }} -b ${{ secrets.COS_BUCKET }} -r ${{ secrets.COS_REGION }}

      - name: 上传到腾讯云 COS
        run: |
          coscmd upload -r .vuepress/dist/ /
      # - name: 部署到腾讯云 COS
      #   uses: TencentCloud/cos-action@v1
      #   with:
      #     secret_id: ${{ secrets.TENCENT_SECRET_ID }}
      #     secret_key: ${{ secrets.TENCENT_SECRET_KEY }}
      #     cos_bucket: ${{ secrets.COS_BUCKET }}
      #     cos_region: ${{ secrets.COS_REGION }}
      #     local_path: .vuepress/dist/
      #     remote_path:
      #     clean: true # 可选：上传前清空目标路径
```

这里我们使用 腾讯云 CLI 进行文件上传，这里有几个**注意点**：

1. branches 要设置为自己更新的分支名
2. node-version：node 版本需要看一下是否符合你项目的需求
3. coscmd upload -r .vuepress/dist/ /
   1. .vuepress/dist/：代表着你构建产物的路径
   2. /：你要放到存储桶的对应地址

首选我们去 [https://console.cloud.tencent.com/cam/capi](https://console.cloud.tencent.com/cam/capi?from_column=20423&from=20423) 这里获取我们的 腾讯云 cos 服务需要的 TENCENT_SECRET_ID 和 TENCENT_SECRET_KEY，然后还需要：

- COS_BUCKET：桶名称
- COS_REGION：桶地区

获取到这些后，我们需要去 github 配置密钥的地方进行配置。

![](https://blog-1304565468.cos.ap-shanghai.myqcloud.com/work/1743240938073-72d13b83-50a6-4f0f-a0e1-9ed25e015d00.png)

填写密钥信息：

![](https://blog-1304565468.cos.ap-shanghai.myqcloud.com/work/1743244868555-a0edb273-7f37-4a6c-ac07-8f1ae2744e34.png)

我们可以进行一次文件上传，然后会去 Actions 下看到：

![](https://blog-1304565468.cos.ap-shanghai.myqcloud.com/work/1743244927055-63d6345a-c42d-4308-94dc-d6c8f85903cf.png)

现在当我们进行文件更新的时候就会自动进行打包部署，但是成功与否是只能回来看 github action 查看，我希望能通过邮件提醒我，继续：

添加脚本：

```javascript
const nodemailer = require("nodemailer");

// 从命令行参数获取邮箱配置
const [emailUser, emailPass, toEmail] = process.argv.slice(2);
const repoName = process.env.GITHUB_REPOSITORY || "未知仓库";
const runId = process.env.GITHUB_RUN_ID || "未知";
const runUrl = `https://github.com/${repoName}/actions/runs/${runId}`;
const branch = process.env.GITHUB_REF_NAME || "main";

async function sendEmail() {
  // 创建邮件传输器
  const transporter = nodemailer.createTransport({
    service: "qq", // 或其他服务，如 'gmail', '163' 等
    auth: {
      user: emailUser,
      pass: emailPass, // QQ 邮箱需要使用授权码而非密码
    },
  });

  // 设置邮件内容
  const mailOptions = {
    from: emailUser,
    to: toEmail,
    subject: `【构建通知】AI 知识库已成功部署 - ${new Date().toLocaleString()}`,
    html: `
      <div style="font-family: Arial, sans-serif; padding: 20px; max-width: 600px; margin: 0 auto; border: 1px solid #eee; border-radius: 5px;">
        <h2 style="color: #18b566;">✅ 部署成功通知</h2>
        <p>您的 <strong>AI 知识库</strong> 网站已成功构建并部署到腾讯云 COS！</p>
        <ul style="list-style-type: none; padding-left: 0;">
          <li><strong>仓库:</strong> ${repoName}</li>
          <li><strong>分支:</strong> ${branch}</li>
          <li><strong>部署时间:</strong> ${new Date().toLocaleString()}</li>
        </ul>
        <p>
          <a href="${runUrl}" style="background-color: #1a73e8; color: white; padding: 10px 20px; text-decoration: none; border-radius: 4px; display: inline-block; margin-top: 10px;">
            查看构建详情
          </a>
        </p>
        <p style="color: #666; font-size: 0.9em; margin-top: 20px;">
          此邮件由 GitHub Actions 自动发送，请勿回复。
        </p>
      </div>
    `,
  };

  try {
    // 发送邮件
    const info = await transporter.sendMail(mailOptions);
    console.log("邮件发送成功：", info.messageId);
    return true;
  } catch (error) {
    console.error("邮件发送失败：", error);
    return false;
  }
}

// 执行邮件发送
sendEmail()
  .then((success) => process.exit(success ? 0 : 1))
  .catch((err) => {
    console.error(err);
    process.exit(1);
  });
```

yaml 配置添加：

```yaml
        # 新增邮件通知服务
        - name: 安装 nodemailer
        if: success() # 仅在上述步骤成功时执行
        run: npm install nodemailer

      - name: 发送邮件通知
        if: success() # 仅在上述步骤成功时执行
        run: node .github/scripts/send-email.js "${{ secrets.EMAIL_USER }}" "${{ secrets.EMAIL_PASS }}" "${{ secrets.EMAIL_TO }}"
```

最后去 github 添加对应的密钥：

EMAIL_USER: 发件人邮箱地址（如 example@qq.com）

EMAIL_PASS: 邮箱授权码（不是邮箱密码，对于 QQ 邮箱需要在设置中生成授权码）

EMAIL_TO: 收件人邮箱地址
