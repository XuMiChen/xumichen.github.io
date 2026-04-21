---
title: 使用 GitHub Pages + Actions 搭建自动化博客：从原理到实践
date: 2026-04-21 11:00:00
tags:
  - GitHub Pages
  - GitHub Actions
  - CI/CD
  - Hexo
  - 博客搭建
categories:
  - 技术
  - 博客搭建
description: 从 GitHub Pages 入门，逐步理解 GitHub Actions、CI/CD 以及自定义域名，构建一套完整的自动化博客部署体系
---

在搭建个人博客（如 Hexo）时，一个核心目标是：

> 实现从代码提交到网站上线的全自动化流程

---

## 1. GitHub Pages：静态站点托管基础

GitHub Pages 是 GitHub 提供的静态资源托管服务，其核心能力是：

- 将仓库中的静态文件（HTML/CSS/JS）发布为网站
- 提供默认访问地址：

https://<username>.github.io

---

## 2. 引入静态生成器：构建步骤的出现

Markdown → HTML（构建过程）→ 发布

---

## 3. GitHub Actions：自动化构建与部署

push 代码
↓
触发 workflow
↓
安装依赖（npm ci）
↓
构建站点（hexo generate）
↓
部署到 GitHub Pages

---

## 4. CI / CD 拆解

CI：源码 → 构建产物  
CD：构建产物 → 线上服务

---

## 5. configure-pages 的角色

可选步骤，用于提供 Pages metadata，不是必须。

---

## 6. Pages 与 Actions 职责

- Actions：构建与部署
- Pages：托管网站

---

## 7. 自定义域名

blog.example.com → 指向 GitHub Pages

---

## 8. 架构

源码 → Actions → Pages → 自定义域名

---

## 总结

GitHub Actions 提供 CI/CD，GitHub Pages 提供托管能力，两者组合实现自动化博客部署。
