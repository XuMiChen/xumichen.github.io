---
title: 从 0 搭建 Hexo 博客 + GitHub Pages 自动部署（完整踩坑记录）
date: 2026-04-06 17:45:00
tags:
  - Hexo
  - GitHub Pages
  - CI/CD
  - 博客搭建
categories:
  - 技术
---

# 从 0 搭建 Hexo 博客 + GitHub Pages 自动部署（完整踩坑记录）

这篇文章记录了我从零开始搭建 Hexo 博客，并使用 GitHub Actions 自动部署的全过程，以及中间踩过的坑和解决方案。

---

# 一、整体目标

最终想实现的是：

写文章 → git push → 自动部署 → 线上博客更新

技术方案：

- Hexo：博客生成器
- GitHub：代码托管
- GitHub Actions：自动构建
- GitHub Pages：托管静态网站

---

# 二、初始化 Hexo

```bash
npm install -g hexo-cli
hexo init my-blog
cd my-blog
npm install
```

本地启动：

```bash
npx hexo server
```

访问：http://localhost:4000

---

# 三、Git 初始化与分支问题

```bash
git init
git add .
git commit -m "init hexo blog"
git branch -M main
```

作用：把默认分支改成 main，避免后续 CI 和 Pages 问题。

---

# 四、npm install 很慢

```bash
npm config set registry https://registry.npmmirror.com
```

---

# 五、npm install 报错

解决：

```bash
rm -rf node_modules
rm -f package-lock.json pnpm-lock.yaml
npm cache clean --force
npm install
```

---

# 六、Git GPG 报错

```bash
git config --global commit.gpgsign false
```

---

# 七、GitHub Actions 自动部署

创建 `.github/workflows/pages.yml`

---

# 八、主题切换（Butterfly）

```bash
git clone https://github.com/jerryc127/hexo-theme-butterfly.git themes/butterfly
```

修改 `_config.yml`：

```yaml
theme: butterfly
```

---

# 九、Pug 渲染问题

```bash
npm install hexo-renderer-pug
```

---

# 十、首页封面优化

创建 `_config.butterfly.yml`：

```yaml
cover:
  index_enable: false
```

---

# 十一、自定义域名

DNS：

CNAME → xumichen.github.io

Hexo：

```yaml
url: https://blog.example.com
root: /
```

---

# 十二、总结

Hexo + GitHub + Actions + Pages = 自动化博客系统 🚀
