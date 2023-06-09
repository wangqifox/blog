---
title: gpu共享调研整理
date: 2022/08/24 08:16:00
---

本文是对GPU共享方案调研之后的简单整理

<!-- more -->

盘点来自工业界的GPU共享方案：[https://zhuanlan.zhihu.com/p/398369404](https://zhuanlan.zhihu.com/p/398369404)

GPU虚拟化，算力隔离，和qGPU：[https://zhuanlan.zhihu.com/p/377073683](https://zhuanlan.zhihu.com/p/377073683)

腾讯 GPUManager虚拟化方案：[https://cloud.tencent.com/developer/article/1685122](https://cloud.tencent.com/developer/article/1685122)

总体来讲，GPU共享没有完美的解决方案。

一类解决方案是Nvidia官方提供的Nvidia MPS，和Nvidia vGPU。Nvidia MPS在容器中应用需要进一步学习，Nvidia vGPU需要使用高端计算卡，不支持消费级游戏显卡，并且只能在虚拟机使用，不支持容器。

另一类是第三方研发的方案。分为两种思路，一种是CUDA劫持。因为CUDA库是公开的，所以这种技术较容易实现。但是CUDA库升级频繁，每当CUDA升级时劫持方案也需要升级。隔离不准确，且无法提供算力精准限制的能力。另一种是内核劫持。因为Nvidia Driver的更新更小，所以适配需求很小，但是研发比较困难。



GPU共享方案目前来看还是各大厂商的核心技术能力，更是有趋动科技这样专门做GPU集群管理的公司。内核劫持的技术方案没有哪家厂商开源其代码，唯一可以使用的是CUDA劫持的方案。


> https://virtaitech.com/product/index
> https://github.com/4paradigm/k8s-device-plugin