---
title: "利用Prometheusalert打造Prometheus告警消息聚合"
date: 2021-09-18T17:22:42+08:00
lastmod: 2021-09-18T17:22:42+08:00
draft: false
description: "利用Prometheusalert打造Prometheus告警消息聚"
tags: ["Prometheusalert", "prometheus", "alertmanager", "k8s"]
categories: ["kubernetes"]
author: "wanzi"

comment: true   # 关闭评论
toc: true       # 关闭文章目录
autoCollapseToc: true
contentCopyright: '<a rel="license noopener" href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank">CC BY-NC-ND 4.0</a>'
reward: true     # 关闭打赏
mathjax: true    # 打开 mathjax
---

## 部署prometheusalert
```shell
git clone https://github.com/feiyu563/PrometheusAlert.git
cd PrometheusAlert/example/helm/prometheusalert
#更新config/app.conf设置登陆用户信息，并配置数据库信息
helm install -n monitoring .
```

## 创建企业微信群机器人
企业微信群以后，点击企业微信群，右键“添加群机器人”，创建群机器人会得到一个机器人的webhook地址，记录该地址备用。

## Prometheus接入配置

### 配置alertmanager
由于我这里prometheus采用https://github.com/prometheus-operator/kube-prometheus.git
这里直接修改alertmanager-secret.yaml配置文件即可，将上面的webhook配置在receivers里即可：
```yaml
apiVersion: v1
data: {}
kind: Secret
type: Opaque
metadata:
  name: alertmanager-main
  namespace: monitoring
stringData:
  alertmanager.yaml: |-
    global:
      resolve_timeout: "5m"
      #email
      smtp_smarthost: 'smtp.exmail.qq.com:465'                   # 邮箱smtp服务器代理
      smtp_from: 'devops_alert@xxx.com'                       # 发送邮箱名称
      smtp_auth_username: 'devops_alert@xxx.com'              # 邮箱名称
      smtp_auth_password: '1DevSop#'                             # 授权密码
      smtp_require_tls: false 		  		 # 不开启tls 默认开启
    inhibit_rules:
    - equal:
      - namespace
      - alertname
      - instance
      source_match:
        severity: critical
      target_match_re:
        severity: warning|info
    - equal:
      - namespace
      - alertname
      - instance
      source_match:
        severity: warning
      target_match_re:
        severity: info
        receivers:
    - name: default
      email_configs:            # 邮箱配置
      - to: "xxxx@163.com"   # 接收警报的email配置
    - name: jf-k8s-node
      webhook_configs:
      - url: "http://palert.con.sdi/prometheusalert?type=wx&tpl=jf-k8s-node&wxurl=https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=0da1f944-9a4e-4221-8036-f322351f7bd8"
```

### 配置Prometheus rules
这里举例磁盘告警规则
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  labels:
    prometheus: k8s
    role: alert-rules
  name: external-node.rules
  namespace: monitoring
spec:
  groups:
  - name: k8s-node.rules
    rules:
    - alert: 磁盘容量
      expr: 100-(node_filesystem_avail_bytes{instance=~'39.100.*',fstype=~"ext4|xfs|fuse.glusterfs"}/node_filesystem_size_bytes {fstype=~"ext4|xfs|fuse.glusterfs"}*100) > 96
      for: 1m
      labels:
        severity: critical
        app: node-alert
      annotations:
        summary: "{{$labels.mountpoint}} 磁盘分区使用率过高！"
        description: "{{$labels.mountpoint }} 磁盘分区使用大于90%(目前使用:{{$value}}%)"
```

## 定制消息模版
对于以上报警规则，默认情况下alertmanger有自己的告警消息模版不是很好用，这里我们统一使用prometheusalert的告警模版进行配置。

根据官方文档，我们新建模版，并指定模版类型为企业微信，模版用途Prometheus，模版内容如下：

```yaml
{{ $var := .externalURL}}{{ range $k,$v:=.alerts }}
{{if eq $v.status "resolved"}}
#### [Prometheus恢复信息]({{$v.generatorURL}})
##### <font color="#FF0000">告警名称</font>：[{{$v.labels.alertname}}]({{$var}})
##### <font color="#FF0000">告警级别</font>：{{$v.labels.severity}}
##### <font color="#FF0000">触发时间</font>：{{GetCSTtime $v.startsAt}}
##### <font color="#02b340">恢复时间</font>：{{GetCSTtime $v.endsAt}}
##### <font color="#FF0000">故障实例</font>：{{$v.labels.instance}}
##### <font color="#FF0000">告警详情</font>：{{$v.annotations.description}}
{{else}}
#### [Prometheus告警信息]({{$v.generatorURL}})
##### <font color="#FF0000">告警名称</font>：[{{$v.labels.alertname}}]({{$var}})
##### <font color="#FF0000">告警级别</font>：{{$v.labels.severity}}
##### <font color="#FF0000">触发时间</font>：{{GetCSTtime $v.startsAt}}
##### <font color="#FF0000">故障实例</font>：{{$v.labels.instance}}
##### <font color="#FF0000">告警详情</font>：{{$v.annotations.description}}
{{end}}
{{ end }}
```

当磁盘空间告警以后，实际效果如下：

![terraform操作流程图](/images/2021/prometheusalert-1.png)

至此，用prometheusalert接受Prometheus告警消息，转发给企业微信群机器人配置完毕。

另外，对于多集群多消息源，prometheusalert的强大就体现出来了。
