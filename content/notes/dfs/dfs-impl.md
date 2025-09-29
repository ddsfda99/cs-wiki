---
title: "在 WSL 复现一个简易分布式文件系统"
date: 2025-09-29
draft: false
tags: ["DFS", "分布式文件系统", "C++", "WSL"]
categories: ["分布式系统", "文件存储"]
---
# 在 WSL 复现一个简易分布式文件系统
> 本文基于开源项目 [Distributed-File-System](https://github.com/vvanirudh/Distributed-File-System) 的学习与复现，项目版权归原作者所有。本文内容仅用于学习与研究目的。  
## 一、背景
分布式文件系统（DFS）是很多大规模存储系统的核心，比如 GFS、HDFS、Ceph。但这些系统太复杂，初学者很难直接看懂。  
这个项目是一个教学级的简化 DFS：用 C++ 实现，代码量小，能帮助我们理解 “文件 → 哈希 → 节点 → 存储” 的基本流程。
## 二、环境准备
WSL (Ubuntu)下复现，进行如下的环境配置：
```bash
sudo apt update
sudo apt install build-essential git -y
git clone https://github.com/vvanirudh/Distributed-File-System.git
cd Distributed-File-System
make
```
会得到两个可执行文件：
* `user`（客户端）
* `node`（存储节点）
## 三、配置文件
项目使用一个 `FileMesh.cfg` 来描述节点信息。我做了一个三节点本地测试版：
```
0,127.0.0.1,5000 ,node0
1,127.0.0.1,5001 ,node1
2,127.0.0.1,5002 ,node2
```
然后建好对应目录：
```bash
mkdir -p node0 node1 node2
```
## 四、运行实验
1. 分别在三个终端运行节点：
```bash
./node 0
./node 1
./node 2
```
2. 新建一个测试文件：
```bash
echo "Hello Distributed File System from WSL!" > test.txt
```
3. 启动客户端（用不冲突的端口，比如 7000）：
```bash
./user 127.0.0.1 7000
```
交互式输入：
```
nodeID → 1
operation → store
filename → test.txt
```
4. 检查结果：
```bash
ls -l node0 node1 node2
```
可以看到 `node0/` 下生成了一个 MD5 值命名的文件，打开它就是 test.txt 原始内容。
## 五、源码解析
**客户端 user.cpp**

客户端/用户程序，负责读配置、向目标节点发起 UDP 请求并建立 TCP 传输，执行 store / retrieve 操作。

**存储节点 node.cpp**

服务端/节点程序，监听并处理来自用户的请求。

**辅助模块**

md5percentile.cpp 核心逻辑是把 128 位的 MD5 值 mod 节点数，实现负载均衡式分配。
## 六、声明
原项目地址：[Distributed-File-System](https://github.com/vvanirudh/Distributed-File-System)

本文仅为个人学习笔记与实验复现，代码版权归原作者所有。


