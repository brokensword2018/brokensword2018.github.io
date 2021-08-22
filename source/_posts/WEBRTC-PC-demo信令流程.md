---
title: WEBRTC PC demo信令流程
date: 2021-08-21 23:05:11
tags:
    - WEBRTC
categories: 音视频
---

本文简单介绍WEBRTC PC官方demo的信令流程

<!-- more -->

## 1. 服务架构
{% asset_img pc_demo架构.bmp pc_demo架构 %}

PC demo的架构如上图所示：
- peerconnection_client A 和 peerconnection_client B是要通信的双方，他们之间建立`PeerConnection`进行音视频数据的传输。
- peerconnection_server 负责在两个client之间的信令传输，包括`SDP`和`ICE`信息。


## 2. 信令流程

{% asset_img pc_demo信令流程.bmp pc_demo信令流程 %}

PC demo的信令流程如上图所示：

1. 发起方创建本地`peerConnection`,`addTrack`。
2. 创建offer, `setLocalDescription`,该过程会启动iceCandidate的收集过程。通过信令通道将offer发送给应答方。
3. 应答方创建`peerConnection`，创建answer,通过信令通道发送给发起方。
4. 发起方和应答方通过信令通道交换iceCandidate。
5. 成功建立连接，进行音视频通话
