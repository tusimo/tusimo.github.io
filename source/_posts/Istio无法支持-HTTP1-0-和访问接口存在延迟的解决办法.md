title: Istio 无法支持 HTTP1.0 和访问接口存在延迟的解决办法
author: tusimo
tags:
  - RKE
  - Rancher
  - Istio
  - HTTP
  - Kubernetes
  - Envoy
categories:
  - Kubernetes
date: 2022-08-10 14:24:00
cover: http://www.designerspics.com/wp-content/uploads/2014/12/paper_plane_free_photo.jpg
---
# 现象
`Rancher` 通过 `RKE` 部署了 `K8S` 集群后,启动了 `Istio`,由于一些项目使用的事 `HTTP/1.0`协议,访问出现问题. `Istio` 默认只支持 `HTTP/1.1` 以上协议版本，并不支持 `HTTP/1.0`。

# 原因

`Istio` 默认不支持 `HTTP1.0` 协议

[Istio 配置参考](https://istio.io/latest/docs/reference/commands/pilot-agent/)


![upload successful](/images/pasted-1.png)

# 解决办法

修改环境变量 `PILOT_HTTP10` 为 1,即可增加对 `HTTP1.0`的支持。


## Rancher 部署的ISTIO



![upload successful](/images/pasted-0.png)

主要方式是修改 环境变量的值.
修改`Deployments` 中 `istiod` 资源的配置.增加环境变量 `PILOT_HTTP10`的值为 1 即可.


![upload successful](/images/pasted-2.png)


# 结果

修改后 `HTTP1.0`已经可以支持了.但是出现了另外的问题.访问接口的时候出现了超时.所有接口访问速度都增加了大概 1 秒的样子.

# 解决 1 秒问题

查阅 `Envoy`的文档后发现,

[Envory相关配置地址](https://www.envoyproxy.io/docs/envoy/v1.23.0/api-v3/extensions/filters/network/http_connection_manager/v3/http_connection_manager.proto.html)

![upload successful](/images/pasted-3.png)

修改改配置为 0 可以解决这个 1 秒延迟的问题.


![upload successful](/images/pasted-4.png)

创建以下 `disable-close-timeout.yaml` 并配置到 kubernetes 集群 : `kubectl apply -f disable-close-timeout.yaml -n istio-system`

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: custom-protocol
  namespace: istio-system
spec:
  configPatches:
  - applyTo: NETWORK_FILTER
    match:
      listener:
        filterChain:
          filter:
            name: envoy.filters.network.http_connection_manager
    patch:
      operation: MERGE
      value:
        name: envoy.filters.network.http_connection_manager
        typed_config:
          '@type': type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          delayed_close_timeout: "0"

```