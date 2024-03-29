我们搭建个人博客的初衷，不就是为了能好好地写文章吗？一切重复且枯燥的工作都应该交给程序去自动完成，尤其是静态博客编译和部署，**我们只需要专注文字**。

## 什么是 GitHub Actions

GitHub Actions 是 GitHub 的持续集成服务。持续集成由很多操作组成，比如抓取代码、运行测试、登录远程服务器，发布到第三方服务等等。GitHub 把这些操作就称为 actions。

很多操作在不同项目里面是类似的，完全可以共享。GitHub 允许开发者把每个操作写成独立的脚本文件，存放到代码仓库，使得其他开发者可以引用。

如果你需要某个 action，不必自己写复杂的脚本，直接引用他人写好的 action 即可，整个持续集成过程，就变成了一个 actions 的组合。这就是 GitHub Actions 最特别的地方。

## 使用 GitHub Actions 自动部署的好处

-   可以直接在线编辑 `md` 文件，立即生效。假设你已发布一篇文章，过几天你在别的电脑上浏览发现有几个明显的错别字，这是完全不能容忍的。但此时你电脑上又没有 `hexo + node.js + git` 等完整的开发环境，重新配置开发环境明显不现实。如果使用 CI，你可以直接用浏览器访问 GitHub 上的项目仓库，直接编辑带错别字的 `md` 文章，改完，在线提交，稍等片刻，你的网站就自动更新了。
    
-   如果手动部署，需要先执行 `hexo g` 编译生成静态文件， 然后推送 `public` 整个文件夹到 GitHub 上，当后期网站文章、图片较多时候，很多时候连接 GitHub 不是那么顺畅，经常要傻等很长的上传时间。使用 GitHub Actions 自动部署，你只需 `push` \_post 文件里单独的 `md` 文件即可，其他不用管，效率瞬间高了许多，其中的好处，谁用谁知道。
    
-   使用 GitHub Actions，你还可以一次性将这些静态博客页面部署到多个服务器上，例如：GitHub Pages、Gitee pages、七牛云、阿里云、腾讯云等等。
    
-   ...
    

## 准备工作

本文以 Hexo + Butterfly 主题搭建博客为例，教你如何使用 GitHub Actions 将博客自动部署到 GitHub Pages。

### 创建 GitHub 仓库

创建两个 [GitHub 仓库](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fnew "https://github.com/new")，一个**公共仓库**和一个**私有仓库**。

-   **私有仓库**用来存储 Hexo 项目源代码。（保证你的重要信息不泄露）
    
    ![](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/others/SCR-20220410-vrk.png)
    
-   **公共仓库**用来存储编译之后的静态页面。
    
    ![](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/others/SCR-20220410-vsg.png)
    

本例：

-   用私有仓库的 `main` 分支来存储项目源代码。
-   用公共仓库的 `main 分支` 来存储静态页面。

当私有仓库的 `main` 有内容 `push` 进来时（例如：主题文件，文章 md 文件、图片等）， 会触发 GitHub Actions 自动编译并部署到公共仓库的 `main 分支`。

### 创建 GitHub Token

创建一个有 **repo** 和 **workflow** 权限的 [GitHub Token](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fsettings%2Ftokens%2Fnew "https://github.com/settings/tokens/new") 。

![](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/others/SCR-20220410-vvq.png)

新生成的 Token 只会显示一次，需要先复制保存，如有遗失，重新生成即可。

### 创建 repository secret

将上面生成的 Token 添加到私有仓库的 `Secrets` 里，并将这个新增的 `secret` 命名为 `HEXO_DEPLOY` （名字无所谓，看你喜欢）。

步骤：私有仓库 -> `settings` -> `Secrets` ->`Actions` ->`New repository secret`。

步骤：私有仓库 -> `settings` -> `Secrets` ->`Dependabot` ->`New repository secret`。

![](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/others/SCR-20220410-vo6.png)

> 新创建的 secret `HEXO_DEPLOY` 在 Actions 配置文件要用到，需跟配置文件保持一致！

### 添加 Actions 配置文件

1.  点击创建workflow：main.yml。

![](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/others/SCR-20220410-vlq.png)

`main.yml` 文件的内容如下：

```xml
name: Deploy My_Blog  #自动化的名称

on:
  # Triggers the workflow on push or pull request events but only for the main branch
  push: # push的时候触发
    branches: [ main ]  # 哪些分支需要触发
  pull_request:  
    branches: [ dev ]

# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  # This workflow contains a single job called "build"
  Blog_CI-CD:
    runs-on: ubuntu-latest  # 服务器环境
    # Steps represent a sequence of tasks that will be executed as part of the job
    
    steps:
      # 检查代码
      - name: Checkout
        uses: actions/checkout@v2  #软件市场的名称
        with: # 参数
          submodules: true
          persist-credentials: false
          
      - name: Setup Node.js
       # 设置 node.js 环境
        uses: actions/setup-node@v1
        with:
          node-version: '12'
          
      - name: Cache node modules
      # 设置包缓存目录，避免每次下载
        uses: actions/cache@v1
        with:
          path: ~/.npm
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
          
      # 配置Hexo环境 
      - name: Setup Hexo
        env:
          ACTION_DEPLOY_KEY: ${{ secrets.HEXO_DEPLOY }}
        run: |
          npm install hexo-cli -g
          npm install
           
      
      # 生成静态文件
      - name: Build
        run: |
          hexo clean 
          hexo g
        
      # 2、部署到 GitHub Pages
      - name: upload GitHub repository
        env: 
          # Github 仓库
          GITHUB_REPO: github.com/zhoujinjianzz/zhoujinjianzz.github.io
         # 将编译后的博客文件推送到指定仓库
        run: |
          cd ./public && git init && git add .
          git config user.name "zhoujinjianzz"       #username改为你github的用户名
          git config user.email "zhoujinjianzz@gmail.com"     #username改为你github的注册邮箱
          git add .
          git commit -m "GitHub Actions Auto Builder at $(date +'%Y-%m-%d %H:%M:%S')"
          git push --force --quiet "https://${{ secrets.HEXO_DEPLOY }}@$GITHUB_REPO" master:main
```

至此，准备工作完毕~

## 自动部署触发流程

1.  修改你的 Hexo 博客源代码（例如：增加文章、修改文章、更改主题、修改主题配置文件等等）。
    
2.  把你修改过的 Hexo 项目内容（只提交修改过的那部分内容） `push` 到 GitHub 私有仓库（本例：keep-site-source）的 `main` 分支。
    
3.  GitHub Actions 检测到 `main` 分支有内容 `push` 进来，会自动执行 action 配置文件的命令，将 Hexo 项目编译成静态页面，然后部署到公共仓库的 `main` 分支。
    
    > `main` 分支是 GitHub Pages 服务的自动生成的分支，你只需在 `main.yml` 文件正确填写，会自动创建。
    
4.  在私有仓库的 Actions 可以查看到你配置的 action ![](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/others/Hexo_Butterfly_Blog_Source.png)
    
5.  通过 GitHub Actions 自动部署成功的效果图： ![image](https://raw.githubusercontent.com/zhoujinjianzz/zhoujinjian.com.images/master/others/zhoujinjianzz.github.io.png)
    

> ❤ 如果对你有帮助，点赞支持下作者~

![](https://lf3-cdn-tos.bytescm.com/obj/static/xitu_juejin_web/00ba359ecd0075e59ffbc3d810af551d.svg) 17

![](https://lf3-cdn-tos.bytescm.com/obj/static/xitu_juejin_web/3d482c7a948bac826e155953b2a28a9e.svg) 收藏

## 参考文档

[如何使用Github+Actions实现Hexo博客自动化部署](https://sujie-168.top/2021/05/24/%E5%A6%82%E4%BD%95%E4%BD%BF%E7%94%A8Github-Actions%E5%AE%9E%E7%8E%B0Hexo%E5%8D%9A%E5%AE%A2%E8%87%AA%E5%8A%A8%E5%8C%96%E9%83%A8%E7%BD%B2/)

[如何使用 GitHub Actions 自动部署 Hexo 博客](https://juejin.cn/post/6943895271751286821)