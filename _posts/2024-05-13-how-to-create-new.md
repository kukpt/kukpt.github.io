---
title: 使用Jekyll Compose 简化创建
date: 2024-05-13 14:30 +0800
tags: [jekyll]
---

## 安装
在项目的Gemfile中添加下列行

```
gem 'jekyll-compose', group: [:jekyll_plugins] 
```
{: file="./Gemfile"}

然后执行
```shell
bundle
```
## 术语
- jekyll [__Pages__](https://jekyllrb.com/docs/pages/) 
- jekyll [__Posts__](https://jekyllrb.com/docs/pages/) 博客文章
- jekyll [__Drafts__](https://jekyllrb.com/docs/posts/#drafts) 草稿

## 使用 Jekyll Compose

### 创建Page
```shell
bundle exec jekyll page "My New Page"
```
### 创建Post
```shell
bundle exec jekyll post "My New Post"
```
### 创建Draft
```shell
bundle exec jekyll draft "My New Draft"
```
### 重命名
```shell
# 重命名草稿
bundle exec jekyll rename _drafts/my-new-draft.md "My Renamed Draft"
# 重命名文章
bundle exec jekyll rename _posts/2024-05-13-my-new-post.md "My Renamed Post"
```
### 发布草稿
```shell
bundle exec  jekyll publish _drafts/my-new-draft.md 
# 指定日期
bundle exec  jekyll publish _drafts/my-new-draft.md --date 2024-05-13
```
### 移除发布
```shell
bundle exec jekyll unpublish _posts/2024-05-13-my-new-post.md
```
## 文档
- [Jekyll::Compose 仓库 > github](https://github.com/jekyll/jekyll-compose)
- [Jekyll 文档](https://jekyllrb.com/docs/)