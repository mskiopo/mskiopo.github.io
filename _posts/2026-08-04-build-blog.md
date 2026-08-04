---
title: 从零搭建个人博客:从 Hexo 迁移到 Jekyll + Chirpy
date: 2026-08-04 15:20:00 +0800
categories: [技术]
tags: [博客搭建, Jekyll, GitHub Pages, 踩坑记录]
---

> 这是我的第一篇正式技术文章,记录从零搭建这个博客的全过程,包括所有踩过的坑,希望能帮到想搭博客的你。

## 为什么搭博客

一直想有个自己的技术博客,需求很明确:

- **免费** —— 不想花钱买服务器
- **可控** —— 内容和数据完全属于自己
- **能长期维护** —— 不需要运维,发布要简单

最终选择了 **静态博客 + GitHub Pages**:完全免费、不需要服务器、`git push` 就能发布。

## 第一版:Hexo

最初用的是 Hexo——很流行的 Node.js 静态博客框架:

```bash
npx hexo init blog
```

搭建还算顺利,但遇到一个奇怪的坑:

> **坑 1:hexo-deployer-git 报 `Spawn failed`**
>
> 部署插件在 Git Bash 环境下报 `Spawn failed`(ENOENT),同样的命令在 CMD 终端里却能正常执行。
>
> 解决:用 CMD 执行部署,或者手动把生成的 `public/` 目录推到 GitHub。

Hexo 本身没问题,但默认主题的风格总觉得不够满意。

## 为什么换到 Jekyll + Chirpy

看到 [huiihao.github.io](https://huiihao.github.io/) 这个博客,一眼就喜欢上了:

- 简洁大气的布局,自带暗色主题
- 文章有目录(TOC)、标签、分类体系
- 阅读体验很好

查了一下,它用的是 **Jekyll + Chirpy 主题**,决定同款!

Jekyll 是 Ruby 写的静态站点生成器,Chirpy 是目前最流行的主题之一,还有官方 starter 模板:

```bash
git clone https://github.com/cotes2020/chirpy-starter.git blog-chirpy
```

> **为什么不怕本地环境?**
>
> Chirpy 有个巨大的优点:**不需要本地安装 Ruby/Jekyll**。写完文章 `git push`,GitHub Actions 在云端自动构建部署。本地只需要 git 和编辑器。

## 配置主题

改 `_config.yml` 里的几个核心配置:

```yaml
lang: zh-CN            # 界面语言
timezone: Asia/Shanghai
title: Sunny 的博客
tagline: 记录技术成长的点滴
url: "https://mskiopo.github.io"
avatar: /assets/img/avatar.jpg
```

头像用了一张自己拍的照片,压缩成 512×512(39KB),清晰又不拖慢页面。

## 踩坑记录:GitHub 篇

### 1. 账号改名

旧用户名又长又乱,改成了 `mskiopo`。注意:**GitHub 用户名改名后,用户主页仓库(`用户名.github.io`)也要跟着改名**,否则页面打不开。改完仓库名,等几分钟域名就生效了。

### 2. `gh` 命令行登录超时

用 `gh auth login` 走浏览器授权,浏览器里明明显示 "Your device is now connected",但 CLI 一直报超时:

```
failed to authenticate via web browser:
read tcp ...: A connection attempt failed...
```

折腾了很久,最后用 **PAT(Personal Access Token)** 搞定:

```bash
echo ghp_你的token | gh auth login --with-token
```

> 注意:给 token 添加 scope 会**重新生成 token 值**,旧 token 立刻失效。之后如果报 "token in keyring is invalid",重新跑一遍上面的命令即可。

### 3. git 全局代理是"死"的

之前配置过 git 全局代理 `127.0.0.1:7890`(Clash)。Clash 没开的时候,这个代理就是死的,push 一直失败:

```
Failed to connect to github.com:443 after 21148 ms
```

解决:push 前删掉代理,推完再恢复:

```bash
git config --global --unset-all http.proxy
git push
git config --global http.proxy http://127.0.0.1:7890  # 需要时再恢复
```

> 网络直连 GitHub 时好时坏,遇到 `Connection was reset` 别慌,重试几次一般都能成功。

### 4. 第一次 push 不触发自动部署

工作流文件第一次推送后,GitHub Actions 没有自动跑。新仓库第一次需要手动触发:

```bash
gh workflow run pages-deploy.yml
```

手动触发一次之后,后续每次 push 都会自动构建部署。

### 5. 小陷阱:管道掩盖了错误

中途有一次用 `git push 2>&1 | tail -1 && echo 成功`,因为 `tail` 会"吃掉" git 的退出码,明明失败了还显示成功。正确的写法:

```bash
out=$(git push 2>&1); st=$?
if [ $st -eq 0 ]; then echo "成功"; else echo "$out"; fi
```

## 小插曲:标签页图标是只"蚂蚁"

主题自带的默认 favicon(浏览器标签页图标)看起来像只蚂蚁,不太好看。我找了一张科幻风格的图片来替换,用 PowerShell 的 System.Drawing 把图片**居中裁剪成正方形**,再缩放到各种尺寸(512 / 180 / 96 / 64 + ICO),覆盖所有浏览器和手机:

```powershell
$src = [System.Drawing.Image]::FromFile('原图.jpg')
# 居中裁剪 + 高质量缩放,输出 favicon-96x96.png、apple-touch-icon.png 等
```

> **坑 6:浏览器会缓存 favicon**
>
> 换完图标部署成功后,浏览器标签页还是显示旧的"蚂蚁"——因为浏览器会激进地缓存图标。
>
> 解决:`Ctrl + F5` 强制刷新;还不行就 F12 → Application → Service Workers → Unregister,再刷新。

## 现在的发布流程

写一篇文章,发布只需要三步:

```bash
# 1. 在 _posts/ 目录新建文件,命名 YYYY-MM-DD-标题.md,头部写好 title/date/categories/tags
# 2. 提交并推送
git add .
git commit -m "新文章:xxx"
git push
# 3. 等 1~2 分钟,GitHub Actions 自动构建部署
```

访问 https://mskiopo.github.io 就能看到新文章,全程不需要本地装任何开发环境。

## 总结

- 静态博客 + GitHub Pages 是个人博客**性价比最高的方案**:免费、快速、可控
- 踩坑不可怕,每个坑都是下一篇博客的素材——这篇博客本身就是证据 😄
- 坚持写下去最重要,先写给自己,再考虑读者

> **写给自己的提醒:不要太贪免费的 API**
>
> 免费的东西都有代价:随时可能停服、限流、数据消失。博客可以搭在免费服务上,但重要的内容一定要有备份、有退路——不要把自己的核心依赖建立在他人的"善意"上。

**种一棵树最好的时间是十年前,其次是现在。写博客也是。**
