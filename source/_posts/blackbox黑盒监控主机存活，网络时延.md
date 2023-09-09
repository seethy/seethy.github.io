---
title: blackbox黑盒监控主机存活，网络时延
date: 2023-09-09 10:46:07
tags:
- blackbox
categories:
- ['组件服务', 'blackbox']
---
    有时候，在一个集群网络中，需要持续关注各个边缘节点的网络性能，以保证能够持续提供高可靠性的服务，因此需要有一个工具能够提供持续探测主机存活，网络时延的机制。

    Blackbox Exporter 是 Prometheus 社区提供的官方黑盒监控解决方案，其允许用户通过：HTTP、HTTPS、DNS、TCP 以及 ICMP 的方式对网络进行探测。本章节主要讲述blackbox ICMP主机探活机制，其本质主要是通过ping命令来检测目的主机的连通性。

**Blackbox配置：**
```yaml
icmp:
  prober: icmp
  timeout: 2s
  icmp:
    preferred_ip_protocol: ip4
    ip_protocol_fallback: true
```

    timeout代表探测的超时时间，重点关注preferred_ip_protocol: ip4配置，因为默认是使用ipv6地址探测，从而导致探测失败。

    Blackbox容器部署配置，因为默认alpine基础镜像普通用户是不支持ping操作的，所以需要加上root用户权限
```yaml
securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  runAsNonRoot: false
  runAsUser: 0
```

    prometheus scrape配置如下， 这里的测试环境中，80.80.130.[79-81]这3个节点的网络是正常的，构造了一个不存在的节点80.80.130.200来测试联通性。
```yaml
- job_name: 'kubernetes-probe-ping-status'
  scheme: https
  scrape_interval: 60s

  tls_config:
    insecure_skip_verify: true
    cert_file: /etc/prometheus/secrets/ztetool-cert/tls.crt
    key_file: /etc/prometheus/secrets/ztetool-cert/tls.key
    ca_file: /etc/prometheus/secrets/ztetool-cert/ca.crt

  static_configs:
    - targets: ['80.80.130.79','80.80.130.80','80.80.130.81','80.80.130.200']
      labels:
        group: icmp
  metrics_path: /probe
  params:
    module: [icmp]
  relabel_configs:
    - action: labelmap
      regex: __meta_kubernetes_pod_label_(.+)
    - source_labels: [__address__]
      target_label: __param_target
    - target_label: __address__
      replacement: blackbox-exporter.ezmon.svc.cluster.local.:6800
    - source_labels: [__param_target]
      target_label: target_url
    - source_labels: [instance]
      target_label: deviceid
      regex: (.*)
      replacement: $1
      action: replace
```
    查看prometheus结果如下， 可以看到正常节点的网络时延，80.80.130.200因为网络不通，所以是2s超时了。
    ![]()
