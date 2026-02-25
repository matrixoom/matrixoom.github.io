---
title: 使用 Hexo + Fluid + GitHub Pages 搭建个人博客（含自动部署 + 评论系统）
cover: /images/cover001.png
sticky: 0
comments: true
categories:
  - 技术
tags:
  - GitHub
---
# 使用 Hexo + Fluid + GitHub Pages 搭建个人博客（含自动部署 + 评论系统）

如果你想搭建一个：

* 免费
* 稳定
* 可长期维护
* 支持评论
* 自动部署

的个人博客，那么这套组合非常合适：

> Hexo + Fluid 主题 + GitHub Pages + GitHub Actions + Giscus

这篇文章会带你完整走一遍流程。

---

# 一、创建 GitHub Pages 仓库

首先注册 GitHub 账号，然后创建一个特殊命名的仓库：

```
<username>.github.io
```

例如：

```
matrixoom.github.io
```

⚠️ 必须用这个命名规则，这是 GitHub Pages 的约定。

---

# 二、本地部署 Hexo（Windows 示例）

## 1️⃣ 安装 Hexo

```bash
npm install -g hexo-cli
```

## 2️⃣ 初始化项目

```bash
hexo init blog
cd blog
npm install
```

## 3️⃣ 启动本地服务

```bash
hexo server
```

浏览器访问：

```
http://localhost:4000
```

如果看到默认页面，说明安装成功。

官方文档参考：
Hexo

---

# 三、安装 Fluid 主题

默认主题较为简单，推荐使用 Fluid。

Fluid 是一个：

* 简洁现代
* 支持评论系统
* 支持暗黑模式
* SEO 友好

的主题。

官方项目：
Fluid

---

## 1️⃣ 安装主题

在 blog 根目录执行：

```bash
git clone https://github.com/fluid-dev/hexo-theme-fluid.git themes/fluid
```

## 2️⃣ 修改站点配置

打开根目录 `_config.yml`：

```yaml
theme: fluid
```

## 3️⃣ 复制主题配置文件

```bash
cp themes/fluid/_config.yml _config.fluid.yml
```

后续推荐修改 `_config.fluid.yml`，不要直接改主题目录里的文件。

---

# 四、添加博客文章

## 1️⃣ 写文章

```bash
hexo new "我的第一篇文章"
```

文件生成在：

```
source/_posts/
```

## 2️⃣ 添加图片

放到：

```
source/images/
```

## 3️⃣ 本地预览

```bash
hexo server
```

---

# 五、传统部署到 GitHub（手动方式）

这种方式需要本地生成再上传。

## 1️⃣ 修改站点配置 `_config.yml`

```yaml
deploy:
  type: git
  repo: https://github.com/YourgithubName/YourgithubName.github.io.git
  branch: main
```

## 2️⃣ 安装部署插件

```bash
npm install hexo-deployer-git --save
```

## 3️⃣ 部署

```bash
hexo clean
hexo generate
hexo deploy
```

过几分钟访问：

```
https://username.github.io
```

---

# 六、推荐方式：使用 GitHub Actions 自动部署

更现代的做法是不上传 public 文件夹，而是：

> 上传源码 → GitHub 自动构建 → 自动部署

---

## 1️⃣ 上传源码

把 blog 文件夹内容 push 到：

```
<username>.github.io
```

⚠️ 确保 `.gitignore` 里包含：

```
public/
```

---

## 2️⃣ 设置 Pages 构建方式

进入仓库：

```
Settings → Pages → Source
```

选择：

```
GitHub Actions
```

---

## 3️⃣ 创建工作流文件

在仓库创建：

```
.github/workflows/pages.yml
```

内容如下（Node 版本改成你的版本）：

```yaml
name: Pages

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Use Node.js 20 # Node版本号
        uses: actions/setup-node@v4
        with:
          node-version: "20" # Node版本号
      - name: Install Dependencies
        run: npm install
      - name: Build
        run: npm run build
      - name: Upload Pages artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public
  deploy:
    needs: build
    permissions:
      pages: write
      id-token: write
    runs-on: ubuntu-latest
    steps:
      - uses: actions/deploy-pages@v4
```

以后你只需要：

```bash
git push
```

博客就会自动更新。

---

# 七、开启评论系统（推荐 Giscus）

因为 GitHub Pages 是静态托管，所以必须使用第三方评论系统。

推荐使用：

Giscus

它基于 GitHub Discussions，稳定且免费。

---

## 1️⃣ 开启 Discussions

进入仓库：

```
Settings → General → Features
```

勾选：

```
Discussions
```

---

## 2️⃣ 安装 Giscus App

访问：

```
https://giscus.app
```

授权你的仓库。

---

## 3️⃣ 配置 Fluid 主题

打开 `_config.fluid.yml`：

启用评论：

```yaml
post:
  comments:
    enable: true
```

设置评论类型：

```yaml
comments:
  type: giscus
```

添加 giscus 配置：

```yaml
giscus:
  repo: username/username.github.io
  repo_id: 你的repo_id
  category: General
  category_id: 你的category_id
  mapping: pathname
  reaction: true
  theme: light
  lang: zh-CN
```

这些 ID 在 giscus 官网生成。

---

## 4️⃣ 推送更新

```bash
git add .
git commit -m "enable giscus"
git push
```

几分钟后评论区就会出现。

---

# 八、自定义域名（可选）

如果你有域名：

在：

```
source/
```

创建文件：

```
CNAME
```

内容写你的域名：

```
www.example.com
```

部署后即可生效。

官方说明参考：
GitHub Pages

---

# 九、最终博客架构

现在你的博客结构是：

```
本地写作
    ↓
git push
    ↓
GitHub Actions 自动构建
    ↓
GitHub Pages 发布
    ↓
Giscus 提供评论
```

完全免费，完全自动化。

---

# 十、总结

你现在拥有：

* 静态博客框架：Hexo
* 现代主题：Fluid
* 免费托管：GitHub Pages
* 自动部署：GitHub Actions
* 评论系统：Giscus

这是一套非常成熟、稳定、适合长期维护的博客方案。


