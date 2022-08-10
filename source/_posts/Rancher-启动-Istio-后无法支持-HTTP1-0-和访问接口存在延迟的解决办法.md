title: Istio 无法支持 HTTP1.0 和访问接口存在延迟的解决办法
author: tusimo
tags:
  - RKE
  - Rancher
  - Istio
  - HTTP
  - Kubernetes
categories:
  - Kubernetes
date: 2022-08-10 14:24:00
---
## 现象
`Rancher` 通过 `RKE` 部署了 `K8S` 集群后,启动了 `Istio`,由于一些项目使用的事 `HTTP/1.0`协议,访问出现问题. `Istio` 默认只支持 `HTTP/1.1` 以上协议版本，并不支持 `HTTP/1.0`。

