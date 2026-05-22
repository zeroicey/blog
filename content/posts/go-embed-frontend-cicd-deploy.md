+++
title = 'Go 项目实战：用 embed 打包前端并通过 GitHub Actions 自动部署'
date = '2026-05-22T00:30:31+08:00'
draft = true
tags = ['Go', 'embed', 'CI/CD', 'GitHub Actions', '前端', '部署', '自动化', '单文件']
description = '这篇文章会记录如何在 Go 项目里使用 embed 将前端构建产物直接打进后端二进制文件，实现单文件交付，并结合 GitHub Actions 串起自动构建、发布和服务器部署流程。'
+++

这篇文章会记录如何在 Go 项目里使用 `embed` 将前端构建产物直接打进后端二进制文件，实现单文件交付；同时再配上 GitHub Actions，把前端构建、Go 编译、制品发布和服务器部署串成一条自动化流水线，让每次提交都能更稳定地完成构建与上线。
