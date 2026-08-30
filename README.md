# Sunny 的博客

个人博客源码,基于 [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) 主题(Jekyll),部署在 GitHub Pages。

线上地址: **https://mskiopo.github.io**

## 写作

在 `_posts/` 下新建 `YYYY-MM-DD-slug.md`(slug 用英文,决定文章 URL),头部写好 front matter 即可:

```yaml
---
title: 文章标题
date: 2026-08-30 12:00:00 +0800
categories: [技术]
tags: [Spring Boot]
pinned: true   # 可选,置顶到首页
---
```

推送到 `main` 分支后,GitHub Actions 会自动构建部署。

## 本地运行

```shell
bundle install
bundle exec jekyll s
```

> 注意:启用自托管资源(`assets.self_host.enabled`)后,需要先初始化子模块:
> `git submodule update --init --recursive`

## 待办

- [x] 开启 giscus 评论(只差最后一步:到 https://github.com/apps/giscus 点 Install 授权本仓库)
- [x] 仓库描述更新
- [ ] 接入 GoatCounter 访问统计(见 `_config.yml` 里 analytics 段)
- [ ] Google/Bing 站长验证(填 `_config.yml` 里 `webmaster_verifications`,然后到对应平台提交 sitemap)
