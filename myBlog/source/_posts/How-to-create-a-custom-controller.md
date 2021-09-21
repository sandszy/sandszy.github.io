---
title: How to create a custom controller
date: 2021-09-19 22:37:05
tags:
---
### 总体步骤

编写自定义控制器代码的过程包括：编写 main 函数、编写自定义控制器的定义，以及编写控制器里的业务逻辑三个部分。

### main函数

main.go/func main

### 控制器定义

controller.go/func NewController

### 业务逻辑

controller.go/func runWorker->func processNextWorkItem->func syncHandler->func updateFooStatus
