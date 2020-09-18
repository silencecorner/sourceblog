---
title: docker muti-stage build
date: 2019-11-04 17:23:04
tags: 
    - CI/CD
    - 运维
categories:
    - 运维
img: /gallery/thumbnails/multi-stage.jpg
---
{% img /gallery/thumbnails/multi-stage.jpg mutistage_build %}
# 前言
docker 多阶段构建是从[17.05.0-ce](https://docs.docker.com/engine/release-notes/#17050-ce)开始支持的，所以你要使用这个功能那么你的docker版本必须大于这个。使用`muti-stage build`可以减少构建之后的镜像大小，使用依赖缓存，可以在CI/CD的场景中使用，避免管理多个编译环境的尴尬（一台机器你打算装多少nodejs、java版本，此外还有编译工具npm、yarn、maven、gradle😅）
