---
title: "制定kubernetes集群备份策略"
date: 2021-09-11T17:22:42+08:00
lastmod: 2021-09-11T17:22:42+08:00
draft: false
description: "制定kubernetes集群备份策略"
tags: ["velero", "k8s", "备份", "keepalived"]
categories: ["kubernetes"]
author: "wanzi"

comment: true   # 关闭评论
toc: true       # 关闭文章目录
autoCollapseToc: true
contentCopyright: '<a rel="license noopener" href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank">CC BY-NC-ND 4.0</a>'
reward: true     # 关闭打赏
mathjax: true    # 打开 mathjax
---


对于备份，每家互联网公司技术人员都要去做的一件事儿，当然我们也不例外，今天我主要针对生产环境kubernetes集群制定一些自己的策略，这里分享给大家。

我这里kubernetes备份目的主要为了防止如下情况出现：
- 防止误操作删除集群中某一个namespace
- 防止误操作导致集群中某一个资源出现异常，比如deployment、configmap等
- 防止误操作删除集群部分资源对象
- 防止etcd数据丢失

## 备份ETCD

备份etcd，防止k8s集群出现了集群级别故障或etcd数据丢失情况，导致整个集群不可用的情况，这种情况只能通过恢复集群恢复业务。

备份etcd脚本如下：

```shell
#!/bin/bash
#ENDPOINTS="https://192.168.1.207:2379,https://192.168.1.208:2379,https://192.168.1.209:2379"
ENDPOINTS="127.0.0.1:2379"
CACERT="/etc/kubernetes/pki/etcd/ca.crt"
CERT="/etc/kubernetes/pki/etcd/server.crt"
KEY="/etc/kubernetes/pki/etcd/server.key"
DATE=`date +%Y%m%d-%H%M%S`
BACKUP_DIR="/home/centos/hostpath/backups/k8s/etcd"
ETCDCTL_API=3 /usr/local/bin/etcdctl --cacert=${CACERT} --cert=${CERT} --key=${KEY}  --endpoints="${ENDPOINTS}" snapshot save ${BACKUP_DIR}/k8s-snapshot-${DATE}.db
find  $BACKUP_DIR/ -type f -mtime +20 -exec rm -f {} \;
```
cron任务计划：

```yaml
50 21 * * * /bin/bash /home/centos/hostpath/backups/k8s/etcdv3-bak.sh
```


## minio对象存储服务搭建

由于我们的存储集群采用的GlusterFS搭建，因此，这里我只能使用minio搭建对象存储，底层采用GlusterFS文件系统；如果你使用阿里云OSS备份你集群资源，请忽略这一步，可以移驾：https://github.com/AliyunContainerService/velero-plugin

由于minio在k8s环境下搭建需要提供对应存储pv/pvc，这里为了简单，直接启动docker方式：

```yaml
version: '2.0'
services:
  minio:
    image: minio/minio:latest
    container_name: minio
    ports:
      - "39000:9000"
      - "39001:9001"
    restart: always
    command: server --console-address ':9001' /data
    environment:
      MINIO_ACCESS_KEY: admin
      MINIO_SECRET_KEY: adminSD#123
    logging:
      options:
        max-size: "1000M" # 最大文件上传限制
        max-file: "100"
      driver: json-file
    volumes:
      - /home/centos/hostpath/backups/k8s/velero:/data # 映射文件路径
    networks:
      - minio

networks:
  minio:
    ipam:
      config:
      - subnet: 10.210.1.0/24
        gateway: 10.210.1.1
```

打开浏览器输入如下地址和账户信息，就可以通过web控制台管理minio对象存储了


```shell
minio web：http://192.168.1.214:39001
minio admin：admin
minio admin passwd：adminSD#123
```


## velero备份之安装客户端

```shell
brew install velero
```

创建文件credentials-velero内容如下,方便后边创建velero server端连接对象存储使用：
```yaml
[default]
aws_access_key_id = admin
aws_secret_access_key = adminSD#123
```

## velero备份之k8s部署velero

```shell
# velero install \
    --provider aws \
    --plugins velero/velero-plugin-for-aws:v1.2.0 \
    --bucket k8s-jf \
    --secret-file ./credentials-velero \
    --use-volume-snapshots=false \
    --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://192.168.1.214:39000
```

## velero备份之velero命令

### 备份,查看,删除操作
```shell
#备份集群ingress-nginx namespace下资源:
velero backup create ingress-nginx-backup --include-namespaces ingress-nginx

#查看备份结果
velero backup describe ingress-nginx-backup
velero backup logs ingress-nginx-backup

#删除备份
velero delete backup ingress-nginx-backup

#备份非ingress-nginx和test命名空间下的资源：
velero backup create k8s-full-test-backup --exclude-namespaces ingress-nginx,test

#备份特定资源类型
velero backup create kube-system-backup --include-resources pod,secret

#--confirm 直接删除备份，无需确认：
velero backup delete kube-system-backup --confirm

#备份带pv pod
velero backup create pvc-backup  --snapshot-volumes --include-namespaces test-velero
```

注意：
- --include-resources备份指定资源类型, --exclude-resources指定排除某些资源类型

### 定时备份：
```shell
# 每六个小时备份一次
velero create schedule ${SCHEDULE_NAME} --schedule="0 */6 * * *"

# 每六个小时使用every备份一次
velero create schedule ${SCHEDULE_NAME} --schedule="@every 6h"

# 创建一个web命名空间的天备份
velero create schedule ${SCHEDULE_NAME} --schedule="@every 24h" --include-namespaces web

# 创建一个周备份，持续时间为90天。
velero create schedule ${SCHEDULE_NAME} --schedule="@every 168h" --ttl 2160h0m0s
```

说明：--ttl可以指定backup的生存周期，在ttl超时后，backup会被定期清理,ttl默认30天


### 备份恢复：

```shell
#从backup创建restore
velero restore create ${RESTORE_NAME} --from-backup ${BACKUP_NAME}

# 从backup创建restore，restore默认名为 ${BACKUP_NAME}-<timestamp>
velero restore create --from-backup ${BACKUP_NAME}

# 从schedule最新一次的backup创建restore
velero restore create --from-schedule ${SCHEDULE_NAME}

# 指定backup中的某些资源创建restore
velero restore create --from-backup backup-2 --include-resources pod,secret

# 恢复集群所有备份，（对已经存在的服务不会覆盖）
velero restore create --from-backup all-ns-backup

# 仅恢复default nginx-example命名空间
velero restore create --from-backup all-ns-backup --include-namespaces default,nginx-example 

# 将test-velero 命名空间资源恢复到test-velero-1下面
velero restore create restore-for-test --from-backup everyday-1-20210203131802 --namespace-mappings test-velero:test-velero-1
```

查看备份
```shell
velero  get  backup   #备份查看
velero  get  schedule #查看定时备份
velero  get  restore  #查看已有的恢复
velero  get  plugins  #查看插件
```

说明：

velero restore create RESTORE_NAME --from-backup BACKUP_NAME --namespace-mappings old-ns-1:new-ns-1,old-ns-2:new-ns-2

Velero可以将资源还原到与其备份来源不同的命名空间中。为此，请使用--namespace-mappings标志


## velero备份实战

### velero对集群进行一次完整备份：
```shell
velero backup create  k8s-jf-test-all
```

### 设置4小时定时备份一次，备份保留2个月：
```shell
# velero create schedule k8s-jf-cron-4h --exclude-namespaces test,tt --schedule="@every 4h" --ttl 1440h
Schedule "k8s-jf-cron-4h" created successfully.
```

### 同集群下，手动从完整备份中恢复其中一个namespace到指定namespace
```shell
# velero restore create  k8s-jf-test-all-restore --from-backup  k8s-jf-test-all --include-namespaces test --namespace-mappings  test:test10000
# velero restore describe k8s-jf-test-all-restore #查看恢复状态
Name:         k8s-jf-test-all-restore
Namespace:    velero
Labels:       <none>
Annotations:  <none>
Phase:                                 InProgress
Estimated total items to be restored:  141
Items restored so far:                 123
Started:    2021-09-07 10:47:44 +0800 CST
Completed:  <n/a>
Backup:  k8s-jf-test-all
Namespaces:
  Included:  all namespaces found in the backup
  Excluded:  <none>
Resources:
  Included:        *
  Excluded:        nodes, events, events.events.k8s.io, backups.velero.io, restores.velero.io, resticrepositories.velero.io
  Cluster-scoped:  auto
Namespace mappings:  test=test10000
Label selector:  <none>
Restore PVs:  auto
Preserve Service NodePorts:  auto
# velero restore get
NAME                      BACKUP            STATUS       STARTED                         COMPLETED   ERRORS   WARNINGS   CREATED                         SELECTOR
k8s-jf-test-all-restore   k8s-jf-test-all   InProgress   2021-09-07 10:47:44 +0800 CST   <nil>       0        0          2021-09-07 10:47:44 +0800 CST   <none>
```

### 跨集群，将定期备份数据恢复
对于velero，跨集群需要保持2个集群使用的是同一个云厂商持久卷方案，这里我们统一使用minio，bucket都使用k8s-jf
```shell
velero install \
    --provider aws \
    --plugins velero/velero-plugin-for-aws:v1.2.0 \
    --bucket k8s-jf \
    --secret-file ./credentials-velero \
    --use-volume-snapshots=false \
    --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://192.168.1.214:39000
```

### 查看已经备份数据：
```shell
#velero backup get
NAME                                STATUS            ERRORS   WARNINGS   CREATED                         EXPIRES   STORAGE LOCATION   SELECTOR
k8s-jf-all-cron-4h-20210907061716   Completed         0        0          2021-09-07 14:17:16 +0800 CST   59d       default            <none>
k8s-jf-all-cron-4h-20210907021627   Completed         0        0          2021-09-07 10:16:27 +0800 CST   59d       default            <none>
k8s-jf-test-all                     Completed         0        0          2021-09-07 10:19:45 +0800 CST   29d       default            <none>
```

### 恢复指定命名空间数据

这里从k8s-jf-all-cron-4h-20210907061716这个备份恢复argocd命名空间数据到本集群argocd-dev下
```shell
# velero restore create  --from-backup  k8s-jf-all-cron-4h-20210907061716 --include-namespaces argocd --namespace-mappings  argocd:argocd-dev
Restore request "k8s-jf-all-cron-4h-20210907061716-20210907155450" submitted successfully.
Run `velero restore describe k8s-jf-all-cron-4h-20210907061716-20210907155450` or `velero restore logs k8s-jf-all-cron-4h-20210907061716-20210907155450` for more details.
# velero restore get
NAME                                               BACKUP                              STATUS       STARTED                         COMPLETED   ERRORS   WARNINGS   CREATED                         SELECTOR
k8s-jf-all-cron-4h-20210907061716-20210907155450   k8s-jf-all-cron-4h-20210907061716   InProgress   2021-09-07 15:54:51 +0800 CST   <nil>       0        0          2021-09-07 15:54:51 +0800 CST   <none>
# velero restore logs k8s-jf-all-cron-4h-20210907061716-20210907161119
time="2021-09-07T08:11:23Z" level=info msg="Attempting to restore Secret: argocd-application-controller-token-wv62v" logSource="pkg/restore/restore.go:1238" restore=velero/k8s-jf-all-cron-4h-20210907061716-20210907161119
time="2021-09-07T08:11:23Z" level=info msg="Restored 2 items out of an estimated total of 61 (estimate will change throughout the restore)" logSource="pkg/restore/restore.go:664" name=argocd-application-controller-token-wv62v namespace=argocd-dev progress= resource=secrets restore=velero/k8s-jf-all-cron-4h-20210907061716-20210907161119
time="2021-09-07T08:11:23Z" level=info msg="Attempting to restore Secret: argocd-dex-server-token-9n4rs" logSource="pkg/restore/restore.go:1238" restore=velero/k8s-jf-all-cron-4h-20210907061716-20210907161119
time="2021-09-07T08:11:23Z" level=info msg="Restored 3 items out of an estimated total of 61 (estimate will change throughout the restore)" logSource="pkg/restore/restore.go:664" name=argocd-dex-server-token-9n4rs namespace=argocd-dev progress= resource=secrets restore=velero/k8s-jf-all-cron-4h-20210907061716-20210907161119
time="2021-09-07T08:11:23Z" level=info msg="Attempting to restore Secret: argocd-secret" logSource="pkg/restore/restore.go:1238" restore=velero/k8s-jf-all-cron-4h-20210907061716-20210907161119
time="2021-09-07T08:11:23Z" level=info msg="Restored 4 items out of an estimated total of 61 (estimate will change throughout the restore)" logSource="pkg/restore/restore.go:664" name=argocd-secret namespace=argocd-dev progress= resource=secrets restore=velero/k8s-jf-all-cron-4h-20210907061716-20210907161119
time="2021-09-07T08:11:23Z" level=info msg="Attempting to restore Secret: argocd-server-token-48vjd" logSource="pkg/restore/restore.go:1238" restore=velero/k8s-jf-all-cron-4h-20210907061716-20210907161119
time="2021-09-07T08:11:23Z" level=info msg="Restored 5 items out of an estimated total of 61 (estimate will change throughout the restore)" logSource="pkg/restore/restore.go:664" name=argocd-server-token-48vjd namespace=argocd-dev progress= resource=secrets restore=velero/k8s-jf-all-cron-4h-20210907061716-20210907161119
time="2021-09-07T08:11:23Z" level=info msg="Attempting to restore Secret: cluster-192.168.1.210-3497337724" logSource="pkg/restore/restore.go:1238" restore=velero/k8s-jf-all-cron-4h-20210907061716-20210907161119
time="2021-09-07T08:11:23Z" level=info msg="Restored 6 items out of an estimated total of 61 (estimate will change throughout the restore)" logSource="pkg/restore/restore.go:664" name=cluster-192.168.1.210-3497337724 namespace=argocd-dev progress= resource=secrets restore=velero/k8s-jf-all-cron-4h-20210907061716-20210907161119
time="2021-09-07T08:11:23Z" level=info msg="Attempting to restore Secret: cluster-192.168.1.214-1096681010" logSource="pkg/restore/restore.go:1238" restore=velero/k8s-jf-all-cron-4h-20210907061716-20210907161119
...
time="2021-09-07T08:11:30Z" level=info msg="Restored 61 items out of an estimated total of 61 (estimate will change throughout the restore)" logSource="pkg/restore/restore.go:664" name=argocd-server namespace=argocd-dev progress= resource=services restore=velero/k8s-jf-all-cron-4h-20210907061716-20210907161119
time="2021-09-07T08:11:30Z" level=info msg="Waiting for all restic restores to complete" logSource="pkg/restore/restore.go:546" restore=velero/k8s-jf-all-cron-4h-20210907061716-20210907161119
time="2021-09-07T08:11:30Z" level=info msg="Done waiting for all restic restores to complete" logSource="pkg/restore/restore.go:562" restore=velero/k8s-jf-all-cron-4h-20210907061716-20210907161119
time="2021-09-07T08:11:30Z" level=info msg="Waiting for all post-restore-exec hooks to complete" logSource="pkg/restore/restore.go:566" restore=velero/k8s-jf-all-cron-4h-20210907061716-20210907161119
time="2021-09-07T08:11:30Z" level=info msg="Done waiting for all post-restore exec hooks to complete" logSource="pkg/restore/restore.go:574" restore=velero/k8s-jf-all-cron-4h-20210907061716-20210907161119
time="2021-09-07T08:11:30Z" level=info msg="restore completed" logSource="pkg/controller/restore_controller.go:480" restore=velero/k8s-jf-all-cron-4h-20210907061716-20210907161119

```

通过日志可以看到原集群argo下数据已经恢复到现在集群argo-dev下

### 卸载velero

卸载velero，注意这里的uninstall不会删除namespace：
```shell
# velero uninstall
You are about to uninstall Velero.
Are you sure you want to continue (Y/N)? y
Velero uninstalled ⛵
# kubectl delete namespace/velero clusterrolebinding/velero
# kubectl delete crds -l component=velero
```

### 备份中遇到异常

```yaml
time="2021-09-07T07:22:35Z" level=info msg="Validating backup storage location" backup-storage-location=default controller=backup-storage-location logSource="pkg/controller/backup_storage_location_controller.go:114"
time="2021-09-07T07:22:36Z" level=info msg="Backup storage location is invalid, marking as unavailable" backup-storage-location=default controller=backup-storage-location logSource="pkg/controller/backup_storage_location_controller.go:117"
time="2021-09-07T07:22:36Z" level=error msg="Error listing backups in backup store" backupLocation=default controller=backup-sync error="rpc error: code = Unknown desc = RequestError: send request failed\ncaused by: Get http://minio.velero.svc:9000/velero?delimiter=%2F&list-type=2&prefix=backups%2F: dial tcp: lookup minio.velero.svc on 10.96.0.10:53: no such host" error.file="/go/src/velero-plugin-for-aws/velero-plugin-for-aws/object_store.go:361" error.function="main.(*ObjectStore).ListCommonPrefixes" logSource="pkg/controller/backup_sync_controller.go:182"
time="2021-09-07T07:22:36Z" level=error msg="Current backup storage locations available/unavailable/unknown: 0/1/0, Backup storage location \"default\" is unavailable: rpc error: code = Unknown desc = RequestError: send request failed\ncaused by: Get http://minio.velero.svc:9000/velero?delimiter=%2F&list-type=2&prefix=: dial tcp: lookup minio.velero.svc on 10.96.0.10:53: no such host)" controller=backup-storage-location logSource="pkg/controller/backup_storage_location_controller.go:164"
time="2021-09-07T07:22:36Z" level=error msg="Current backup storage locations available/unavailable/unknown: 0/1/0)" controller=backup-storage-location logSource="pkg/controller/backup_storage_location_controller.go:166"
```

说明CRD资源BackupStorageLocation信息以及对象存储账户和自己设置的不一致，由于重新安装没有清理干净旧的资源信息


这里重新安装即可：
```shell
# velero uninstall
You are about to uninstall Velero.
Are you sure you want to continue (Y/N)? y
Velero uninstalled ⛵
# kubectl delete namespace/velero clusterrolebinding/velero
# kubectl delete crds -l component=velero
# velero install \
    --provider aws \
    --plugins velero/velero-plugin-for-aws:v1.2.0 \
    --bucket k8s-jf \
    --secret-file ./credentials-velero \
    --use-volume-snapshots=false \
    --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://192.168.1.214:39000
# kubectl -n velero get backupstoragelocation default -o yaml #查看资源信息已经是自己想要的
apiVersion: velero.io/v1
kind: BackupStorageLocation
metadata:
  creationTimestamp: "2021-09-07T07:47:44Z"
  generation: 1
  labels:
    component: velero
  name: default
  namespace: velero
  resourceVersion: "1184696"
  selfLink: /apis/velero.io/v1/namespaces/velero/backupstoragelocations/default
  uid: 39502e43-272e-461f-a114-a9ec955f0510
spec:
  config:
    region: minio
    s3ForcePathStyle: "true"
    s3Url: http://192.168.1.214:39000
  default: true
  objectStorage:
    bucket: k8s-jf
  provider: aws
status:
  lastSyncedTime: "2021-09-07T07:50:00Z"
  lastValidationTime: "2021-09-07T07:50:00Z"
  phase: Available            
```

