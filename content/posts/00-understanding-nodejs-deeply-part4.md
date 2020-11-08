---
title: "《深入浅出 Node.js》学习笔记 Part4"
date: 2019-01-18T21:41:00+08:00
description: "测试，产品化"
draft: false
tags: [Node]
---

# 第 10 章 测试

略。

# 第 11 章 产品化

项目工程化：
- 目录结构 [Project Structure Practices](https://github.com/goldbergyoni/nodebestpractices#1-project-structure-practices)
- 构建工具 Grunt
- 编码规范 ESLint
- 代码审查
  
部署流程

性能：
- 动静分离：Nginx CDN
- 启用缓存：Redis
- 多进程架构：官方 process_child cluster 模块，社区的 pm2 模块进行进程管理
- 读写分离：数据库读写分离
  
日志：
- winston 库

监控报警：
- 日志监控
- 响应时间
- 进程监控
- 磁盘监控
- 内存监控
- CPU 占用监控
- CPU load 监控
- I/O 负载
- 网络监控
- 应用状态监控
- DNS 监控

报警：
- 邮件报警：nodemailer 模块发送邮件
- 短信或电话报警

稳定性：
多进程、多机器、多机房，分布式设计。
容灾备份

异构共存：
协议