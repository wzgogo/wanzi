---
title: "terraform安装与命令详解"
date: 2021-02-25T17:22:42+08:00
lastmod: 2021-02-25T17:22:42+08:00
draft: false
description: "terraform安装与命令详解"
tags: ["terraform", "aliyun"]
categories: ["devops"]
author: "wanzi"

comment: true   # 关闭评论
toc: true       # 关闭文章目录
autoCollapseToc: true
contentCopyright: '<a rel="license noopener" href="https://creativecommons.org/licenses/by-nc-nd/4.0/" target="_blank">CC BY-NC-ND 4.0</a>'
reward: true     # 关闭打赏
mathjax: true    # 打开 mathjax
---

## 安装Terraform

### Mac系统安装
```shell
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
```
### Linux系统安装
1. ubuntu安装
```shell
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt-get update && sudo apt-get install terraform
```

2. centos系统
```shell
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
sudo yum -y install terraform
```
### 验证安装
```shell
# terraform -v
Terraform v0.14.3

Your version of Terraform is out of date! The latest version
is 0.14.7. You can update by downloading from https://www.terraform.io/downloads.html
# terraform
Usage: terraform [global options] <subcommand> [args]

The available commands for execution are listed below.
The primary workflow commands are given first, followed by
less common or more advanced commands.

Main commands:
  init          Prepare your working directory for other commands
  validate      Check whether the configuration is valid
  plan          Show changes required by the current configuration
  apply         Create or update infrastructure
  destroy       Destroy previously-created infrastructure

All other commands:
  console       Try Terraform expressions at an interactive command prompt
  fmt           Reformat your configuration in the standard style
  force-unlock  Release a stuck lock on the current workspace
  get           Install or upgrade remote Terraform modules
  graph         Generate a Graphviz graph of the steps in an operation
  import        Associate existing infrastructure with a Terraform resource
  login         Obtain and save credentials for a remote host
  logout        Remove locally-stored credentials for a remote host
  output        Show output values from your root module
  providers     Show the providers required for this configuration
  refresh       Update the state to match remote systems
  show          Show the current state or a saved plan
  state         Advanced state management
  taint         Mark a resource instance as not fully functional
  untaint       Remove the 'tainted' state from a resource instance
  version       Show the current Terraform version
  workspace     Workspace management

Global options (use these before the subcommand, if any):
  -chdir=DIR    Switch to a different working directory before executing the
                given subcommand.
  -help         Show this help output, or the help for a specified subcommand.
  -version      An alias for the "version" subcommand.
```

## terraform命令之资源管理

###   资源初始化
对于一个terraform资源项目，我这里创建了3个基本文件，分别为：main.tf（入口文件），variables.tf（变量信息），versions.tf（版本信息）
```shell
# ls 
main.tf     variables.tf      versions.tf
# terraform init

Initializing the backend...

Initializing provider plugins...
- Reusing previous version of aliyun/alicloud from the dependency lock file
- Using aliyun/alicloud v1.115.1 from the shared cache directory

Terraform has been successfully initialized!
```
###  格式化terraform文件
fmt默认会回格式化处理当前目录下.tf文件，并格式为标准的tf格式。
```shell
# terraform fmt 
main.tf
variables.tf
versions.tf
# terraform fmt -diff  #格式化处理
main.tf
--- old/main.tf
+++ new/main.tf
@@ -1,7 +1,7 @@
 provider "alicloud" {
   region     = var.region
   access_key = var.alicloud_access_key
-  secret_key =  var.alicloud_secret_key
+  secret_key = var.alicloud_secret_key
 }

 resource "alicloud_vpc" "vpc" {
@@ -12,7 +12,7 @@
 resource "alicloud_vswitch" "vsw" {
   vpc_id            = alicloud_vpc.vpc.id
   cidr_block        = "10.100.0.0/24"
-  availability_zone =  var.availability_zone
+  availability_zone = var.availability_zone
 }

 resource "alicloud_security_group" "default" {
variables.tf
--- old/variables.tf
+++ new/variables.tf
@@ -4,7 +4,7 @@
 }

 variable "alicloud_secret_key" {
-  default                     = "4Z4gbl3d9TGz9jWobv9MPwInvyH2Kf"
+  default     = "4Z4gbl3d9TGz9jWobv9MPwInvyH2Kf"
   description = "The Alicloud Access Secret Key to launch resources.  Support to environment 'ALICLOUD_SECRET_KEY'."
 }
```
###  创建资源计划
terraform plan 会检查一组更改的执行计划是否符合您的期望，而不会更改实际资源或状态。
```shell
# terraform  plan

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # alicloud_instance.wanzi_test will be created
  + resource "alicloud_instance" "wanzi_test" {
      + availability_zone             = "cn-hangzhou-i"
      + credit_specification          = (known after apply)
      + deletion_protection           = false
      + dry_run                       = false
      + host_name                     = (known after apply)
      + id                            = (known after apply)
      + image_id                      = "ubuntu_18_04_64_20G_alibase_20190624.vhd"
      + instance_charge_type          = "PostPaid"
      + instance_name                 = "wanzi_tf001"
      + instance_type                 = "ecs.s6-c1m2.small"
      + internet_max_bandwidth_in     = (known after apply)
      + internet_max_bandwidth_out    = 0
      + key_name                      = (known after apply)
      + password                      = (sensitive value)
      + private_ip                    = (known after apply)
      + public_ip                     = (known after apply)
      + role_name                     = (known after apply)
      + security_groups               = (known after apply)
      + spot_strategy                 = "NoSpot"
      + status                        = "Running"
      + subnet_id                     = (known after apply)
      + system_disk_category          = "cloud_efficiency"
      + system_disk_performance_level = (known after apply)
      + system_disk_size              = 40
      + volume_tags                   = (known after apply)
      + vswitch_id                    = (known after apply)
    }
```
###  创建云资源
terraform apply 会自动生成一个资源创建计划，并批准执行该计划，同时在当前目录下会生成tfstate文件
```shell
# terraform apply
An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # alicloud_instance.wanzi_test will be created
  + resource "alicloud_instance" "wanzi_test" {
      + availability_zone             = "cn-hangzhou-i"
      + credit_specification          = (known after apply)
      + deletion_protection           = false
      + dry_run                       = false
      + host_name                     = (known after apply)
      + id                            = (known after apply)
      + image_id                      = "ubuntu_18_04_64_20G_alibase_20190624.vhd"
      + instance_charge_type          = "PostPaid"
      + instance_name                 = "wanzi_tf001"
      + instance_type                 = "ecs.s6-c1m2.small"
      + internet_max_bandwidth_in     = (known after apply)
      + internet_max_bandwidth_out    = 0
      + key_name                      = (known after apply)
      + password                      = (sensitive value)
      + private_ip                    = (known after apply)
      + public_ip                     = (known after apply)
      + role_name                     = (known after apply)
      + security_groups               = (known after apply)
      + spot_strategy                 = "NoSpot"
      + status                        = "Running"
      + subnet_id                     = (known after apply)
      + system_disk_category          = "cloud_efficiency"
      + system_disk_performance_level = (known after apply)
      + system_disk_size              = 40
      + volume_tags                   = (known after apply)
      + vswitch_id                    = (known after apply)
    }

  # alicloud_security_group.default will be created
  + resource "alicloud_security_group" "default" {
      + id                  = (known after apply)
      + inner_access        = (known after apply)
      + inner_access_policy = (known after apply)
      + name                = "default"
      + security_group_type = "normal"
      + vpc_id              = (known after apply)
    }
......
......
Plan: 5 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

alicloud_vpc.vpc: Creating...
alicloud_vpc.vpc: Creation complete after 9s [id=vpc-bp1kulcyygsi727aay4hd]
alicloud_security_group.default: Creating...
alicloud_vswitch.vsw: Creating...
alicloud_security_group.default: Creation complete after 1s [id=sg-bp11s5pka9pxtj6pn4xq]
alicloud_security_group_rule.allow_all_tcp: Creating...
alicloud_security_group_rule.allow_all_tcp: Creation complete after 1s [id=sg-bp11s5pka9pxtj6pn4xq:ingress:tcp:1/65535:intranet:0.0.0.0/0:accept:1]
alicloud_vswitch.vsw: Creation complete after 4s [id=vsw-bp1wgpgz9z8y2lfsl2beo]
alicloud_instance.wanzi_test: Creating...
alicloud_instance.wanzi_test: Still creating... [10s elapsed]
alicloud_instance.wanzi_test: Still creating... [20s elapsed]
alicloud_instance.wanzi_test: Creation complete after 22s [id=i-bp1gt9mb9asadff9r2zr]

Apply complete! Resources: 5 added, 0 changed, 0 destroyed.
```

###  查看创建的资源信息

terraform show 会查看当前项目创建了哪些资源数据，

terraform show -json  以json形式查看数据

```shell
# terraform show
# alicloud_instance.wanzi_test:
resource "alicloud_instance" "wanzi_test" {
    availability_zone          = "cn-hangzhou-i"
    deletion_protection        = false
    dry_run                    = false
    host_name                  = "iZbp1gt9mb9asadff9r2zrZ"
    id                         = "i-bp1gt9mb9asadff9r2zr"
    image_id                   = "ubuntu_18_04_64_20G_alibase_20190624.vhd"
    instance_charge_type       = "PostPaid"
    instance_name              = "wanzi_tf001"
    instance_type              = "ecs.s6-c1m2.small"
    internet_charge_type       = "PayByTraffic"
    internet_max_bandwidth_in  = -1
    internet_max_bandwidth_out = 0
    password                   = (sensitive value)
    private_ip                 = "10.100.0.234"
    security_groups            = [
        "sg-bp11s5pka9pxtj6pn4xq",
    ]
    spot_price_limit           = 0
    spot_strategy              = "NoSpot"
    status                     = "Running"
    subnet_id                  = "vsw-bp1wgpgz9z8y2lfsl2beo"
    system_disk_category       = "cloud_efficiency"
    system_disk_size           = 40
    volume_tags                = {}
    vswitch_id                 = "vsw-bp1wgpgz9z8y2lfsl2beo"
}

# alicloud_security_group.default:
resource "alicloud_security_group" "default" {
    id                  = "sg-bp11s5pka9pxtj6pn4xq"
    inner_access        = true
    inner_access_policy = "Accept"
    name                = "default"
    security_group_type = "normal"
    vpc_id              = "vpc-bp1kulcyygsi727aay4hd"
}

# alicloud_security_group_rule.allow_all_tcp:
resource "alicloud_security_group_rule" "allow_all_tcp" {
    cidr_ip           = "0.0.0.0/0"
    id                = "sg-bp11s5pka9pxtj6pn4xq:ingress:tcp:1/65535:intranet:0.0.0.0/0:accept:1"
    ip_protocol       = "tcp"
    nic_type          = "intranet"
    policy            = "accept"
    port_range        = "1/65535"
    priority          = 1
    security_group_id = "sg-bp11s5pka9pxtj6pn4xq"
    type              = "ingress"
}

# alicloud_vpc.vpc:
resource "alicloud_vpc" "vpc" {
    cidr_block        = "10.100.0.0/16"
    id                = "vpc-bp1kulcyygsi727aay4hd"
    name              = "tf_test_foo"
    resource_group_id = "rg-acfm2ogp24u3rcy"
    route_table_id    = "vtb-bp1wy8srerq12rta02r03"
    router_id         = "vrt-bp1apvobefvhshksnnwvm"
    router_table_id   = "vtb-bp1wy8srerq12rta02r03"
}

# alicloud_vswitch.vsw:
resource "alicloud_vswitch" "vsw" {
    availability_zone = "cn-hangzhou-i"
    cidr_block        = "10.100.0.0/24"
    id                = "vsw-bp1wgpgz9z8y2lfsl2beo"
    vpc_id            = "vpc-bp1kulcyygsi727aay4hd"
}
```

###  标记污点
terrraform taint 命令用于把某个资源标记为“被污染”状态，当再次执行 apply 命令时，这个被污染的资源将会被先释放，然后再创建一个新的，相当于对这个特定资源做了先删除后新建的操作。

```shell
# terraform taint alicloud_instance.wanzi_test
Resource instance alicloud_instance.wanzi_test has been marked as tainted.
```
而terrraform untaint正好相反，用于取消“被污染”标记，使其恢复到正常的状态。
```shell
# terraform untaint alicloud_instance.wanzi_test
Resource instance alicloud_instance.wanzi_test has been successfully untainted.
```

###  销毁云资源数据
terraform destory 将根据当前资源配置，销毁云端资源数据
```shell
#terraform destroy

Plan: 0 to add, 0 to change, 5 to destroy.

Do you really want to destroy all resources?
  Terraform will destroy all your managed infrastructure, as shown above.
  There is no undo. Only 'yes' will be accepted to confirm.

  Enter a value: yes

alicloud_security_group_rule.allow_all_tcp: Destroying... [id=sg-bp10tup89oothxz8tny1:ingress:tcp:1/65535:intranet:0.0.0.0/0:accept:1]
alicloud_instance.wanzi_test: Destroying... [id=i-bp10ukz4nlr894mhebgl]
alicloud_security_group_rule.allow_all_tcp: Destruction complete after 0s
alicloud_instance.wanzi_test: Still destroying... [id=i-bp10ukz4nlr894mhebgl, 10s elapsed]
alicloud_instance.wanzi_test: Still destroying... [id=i-bp10ukz4nlr894mhebgl, 20s elapsed]
alicloud_instance.wanzi_test: Destruction complete after 28s
alicloud_security_group.default: Destroying... [id=sg-bp10tup89oothxz8tny1]
alicloud_vswitch.vsw: Destroying... [id=vsw-bp1ap7ccst3fjxnw4pnza]
alicloud_security_group.default: Destruction complete after 9s
alicloud_vswitch.vsw: Still destroying... [id=vsw-bp1ap7ccst3fjxnw4pnza, 10s elapsed]
alicloud_vswitch.vsw: Destruction complete after 20s
alicloud_vpc.vpc: Destroying... [id=vpc-bp1obwt5ded2i0zlbu052]
alicloud_vpc.vpc: Destruction complete after 3s

Destroy complete! Resources: 5 destroyed.
```
###  将云端数据导入到本地项目
terraform import 通过云端实例ID来生成本地资源数据，本地目录会生成terraform.tfstate文件，对于本地项目已存在数据的导入前请先备份tfstate文件和.terraform目录；对于已经导入到本地的数据，可以通过terraform show展示出terrafrom文件格式，copy出来并进一步处理，即可得到tf资源文件内容。
```shell
# cat yunduan.tf
resource "alicloud_instance" "test999" {
  # (resource arguments)
}
#
# terraform import alicloud_instance.test999 i-bp1etiv4002h9q27lb97
alicloud_instance.test999: Importing from ID "i-bp1etiv4002h9q27lb97"...
alicloud_instance.test999: Import prepared!
  Prepared alicloud_instance for import
alicloud_instance.test999: Refreshing state... [id=i-bp1etiv4002h9q27lb97]

Import successful!

The resources that were imported are shown above. These resources are now in
your Terraform state and will henceforth be managed by Terraform.
# cat  terraform.tfstate
{
  "version": 4,
  "terraform_version": "0.14.3",
  "serial": 1,
  "lineage": "779fad5e-b076-8cfd-6041-f6eef8c88b8a",
  "outputs": {},
  "resources": [
    {
      "mode": "managed",
      "type": "alicloud_instance",
      "name": "test999",
      "provider": "provider[\"registry.terraform.io/aliyun/alicloud\"]",
      "instances": [
        {
          "schema_version": 0,
          "attributes": {
            "allocate_public_ip": null,
            "auto_release_time": "",
            "auto_renew_period": null,
            "availability_zone": "cn-hangzhou-i",
            "credit_specification": "",
            "data_disks": [],
            "deletion_protection": false,
            "description": "",
            "dry_run": null,
            "force_delete": null,
            "host_name": "iZbp1etiv4002h9q27lb97Z",
            "id": "i-bp1etiv4002h9q27lb97",
            "image_id": "ubuntu_18_04_64_20G_alibase_20190624.vhd",
            "include_data_disks": null,
            "instance_charge_type": "PostPaid",
            "instance_name": "wanzi_tf001",
            "instance_type": "ecs.s6-c1m2.small",
            "internet_charge_type": "PayByTraffic",
            "internet_max_bandwidth_in": -1,
            "internet_max_bandwidth_out": 0,
            "io_optimized": null,
            "is_outdated": null,
            "key_name": "",
            "kms_encrypted_password": null,
            "kms_encryption_context": null,
            "password": "",
            "period": null,
            "period_unit": null,
            "private_ip": "10.100.0.169",
            "public_ip": "",
            "renewal_status": null,
            "resource_group_id": "",
            "role_name": "",
            "security_enhancement_strategy": null,
            "security_groups": [
              "sg-bp14pij6g7sjmn9bz92a"
            ],
            "spot_price_limit": 0,
            "spot_strategy": "NoSpot",
            "status": "Running",
            "subnet_id": "vsw-bp1c966jdtiw1qwh2tng8",
            "system_disk_auto_snapshot_policy_id": "",
            "system_disk_category": "cloud_efficiency",
            "system_disk_description": null,
            "system_disk_name": null,
            "system_disk_performance_level": "",
            "system_disk_size": 40,
            "tags": {},
            "timeouts": {
              "create": null,
              "delete": null,
              "update": null
            },
            "user_data": "",
            "volume_tags": {},
            "vswitch_id": "vsw-bp1c966jdtiw1qwh2tng8"
          },
          "sensitive_attributes": [],
          "private": "eyJlMmJmYjczMC1lY2FhLTExZTYtOGY4OC0zNDM2M2JjN2M0YzAiOnsiY3JlYXRlIjo2MDAwMDAwMDAwMDAsImRlbGV0ZSI6MTIwMDAwMDAwMDAwMCwidXBkYXRlIjo2MDAwMDAwMDAwMDB9LCJzY2hlbWFfdmVyc2lvbiI6IjAifQ=="
        }
      ]
    }
  ]
}
# terraform show
# alicloud_instance.test999:
resource "alicloud_instance" "test999" {
    availability_zone          = "cn-hangzhou-i"
    deletion_protection        = false
    host_name                  = "iZbp1etiv4002h9q27lb97Z"
    id                         = "i-bp1etiv4002h9q27lb97"
    image_id                   = "ubuntu_18_04_64_20G_alibase_20190624.vhd"
    instance_charge_type       = "PostPaid"
    instance_name              = "wanzi_tf001"
    instance_type              = "ecs.s6-c1m2.small"
    internet_charge_type       = "PayByTraffic"
    internet_max_bandwidth_in  = -1
    internet_max_bandwidth_out = 0
    private_ip                 = "10.100.0.169"
    security_groups            = [
        "sg-bp14pij6g7sjmn9bz92a",
    ]
    spot_price_limit           = 0
    spot_strategy              = "NoSpot"
    status                     = "Running"
    subnet_id                  = "vsw-bp1c966jdtiw1qwh2tng8"
    system_disk_category       = "cloud_efficiency"
    system_disk_size           = 40
    tags                       = {}
    volume_tags                = {}
    vswitch_id                 = "vsw-bp1c966jdtiw1qwh2tng8"

    timeouts {}
}
```

###  terraform资源关系绘图
每个模板定义的资源之间都存在不同程度的关系，terraform graph可以绘制资源关系大图，如下：
```shell
# terraform graph
digraph {
        compound = "true"
        newrank = "true"
        subgraph "root" {
                "[root] alicloud_instance.wanzi_test (expand)" [label = "alicloud_instance.wanzi_test", shape = "box"]
                "[root] alicloud_security_group.default (expand)" [label = "alicloud_security_group.default", shape = "box"]
                "[root] alicloud_security_group_rule.allow_all_tcp (expand)" [label = "alicloud_security_group_rule.allow_all_tcp", shape = "box"]
                "[root] alicloud_vpc.vpc (expand)" [label = "alicloud_vpc.vpc", shape = "box"]
                "[root] alicloud_vswitch.vsw (expand)" [label = "alicloud_vswitch.vsw", shape = "box"]
                "[root] provider[\"registry.terraform.io/aliyun/alicloud\"]" [label = "provider[\"registry.terraform.io/aliyun/alicloud\"]", shape = "diamond"]
                "[root] var.alicloud_access_key" [label = "var.alicloud_access_key", shape = "note"]
                "[root] var.alicloud_secret_key" [label = "var.alicloud_secret_key", shape = "note"]
                "[root] var.availability_zone" [label = "var.availability_zone", shape = "note"]
                "[root] var.disk_category" [label = "var.disk_category", shape = "note"]
                "[root] var.disk_size" [label = "var.disk_size", shape = "note"]
                "[root] var.ecs_password" [label = "var.ecs_password", shape = "note"]
                "[root] var.ecs_type" [label = "var.ecs_type", shape = "note"]
                "[root] var.image_id" [label = "var.image_id", shape = "note"]
                "[root] var.internet_charge_type" [label = "var.internet_charge_type", shape = "note"]
                "[root] var.internet_max_bandwidth_out" [label = "var.internet_max_bandwidth_out", shape = "note"]
                "[root] var.region" [label = "var.region", shape = "note"]
                "[root] alicloud_instance.wanzi_test (expand)" -> "[root] alicloud_security_group.default (expand)"
                "[root] alicloud_instance.wanzi_test (expand)" -> "[root] alicloud_vswitch.vsw (expand)"
                "[root] alicloud_instance.wanzi_test (expand)" -> "[root] var.disk_category"
                "[root] alicloud_instance.wanzi_test (expand)" -> "[root] var.ecs_password"
                "[root] alicloud_instance.wanzi_test (expand)" -> "[root] var.ecs_type"
                "[root] alicloud_instance.wanzi_test (expand)" -> "[root] var.image_id"
                "[root] alicloud_security_group.default (expand)" -> "[root] alicloud_vpc.vpc (expand)"
                "[root] alicloud_security_group_rule.allow_all_tcp (expand)" -> "[root] alicloud_security_group.default (expand)"
                "[root] alicloud_vpc.vpc (expand)" -> "[root] provider[\"registry.terraform.io/aliyun/alicloud\"]"
                "[root] alicloud_vswitch.vsw (expand)" -> "[root] alicloud_vpc.vpc (expand)"
                "[root] alicloud_vswitch.vsw (expand)" -> "[root] var.availability_zone"
                "[root] meta.count-boundary (EachMode fixup)" -> "[root] alicloud_instance.wanzi_test (expand)"
                "[root] meta.count-boundary (EachMode fixup)" -> "[root] alicloud_security_group_rule.allow_all_tcp (expand)"
                "[root] meta.count-boundary (EachMode fixup)" -> "[root] var.disk_size"
                "[root] meta.count-boundary (EachMode fixup)" -> "[root] var.internet_charge_type"
                "[root] meta.count-boundary (EachMode fixup)" -> "[root] var.internet_max_bandwidth_out"
                "[root] provider[\"registry.terraform.io/aliyun/alicloud\"] (close)" -> "[root] alicloud_instance.wanzi_test (expand)"
                "[root] provider[\"registry.terraform.io/aliyun/alicloud\"] (close)" -> "[root] alicloud_security_group_rule.allow_all_tcp (expand)"
                "[root] provider[\"registry.terraform.io/aliyun/alicloud\"]" -> "[root] var.alicloud_access_key"
                "[root] provider[\"registry.terraform.io/aliyun/alicloud\"]" -> "[root] var.alicloud_secret_key"
                "[root] provider[\"registry.terraform.io/aliyun/alicloud\"]" -> "[root] var.region"
                "[root] root" -> "[root] meta.count-boundary (EachMode fixup)"
                "[root] root" -> "[root] provider[\"registry.terraform.io/aliyun/alicloud\"] (close)"
        }
}

```
该命令的结果还可以通过命令 terraform graph | dot -Tsvg > graph.svg 直接导出为一张图片（需要提前安装graphviz： brew install graphviz ）
```shell
terraform graph | dot -Tsvg > ~/Downloads/graph.svg
```
查看graph.svg可以看到各个资源之间关系图谱：


## terraform命令之State管理
### 查看当前state里存放所有资源
```shell
# terraform state list
alicloud_instance.wanzi_test
alicloud_security_group.default
alicloud_security_group_rule.allow_all_tcp
alicloud_vpc.vpc
alicloud_vswitch.vsw
```

### 查看某一个resource具体数据
```shell
# terraform state show alicloud_vswitch.vsw
# alicloud_vswitch.vsw:
resource "alicloud_vswitch" "vsw" {
    availability_zone = "cn-hangzhou-i"
    cidr_block        = "10.100.0.0/24"
    id                = "vsw-bp1wgpgz9z8y2lfsl2beo"
    vpc_id            = "vpc-bp1kulcyygsi727aay4hd"
}
```

### 移除特定资源
terraform state rm <资源类型>.<资源名称>
state rm 命令用于将state中的某个资源移除，但是实际上并不会真正删除这个资源，另外也可以通过import操作从云端恢复到本地。
```shell
# terraform state rm  alicloud_security_group.default
Removed alicloud_security_group.default
Successfully removed 1 resource instance(s).
# terraform state list
alicloud_instance.wanzi_test
alicloud_vpc.vpc
alicloud_vswitch.vsw
# terraform import alicloud_security_group.default sg-bp11s5pka9pxtj6pn4xq
alicloud_security_group.default: Importing from ID "sg-bp11s5pka9pxtj6pn4xq"...
alicloud_security_group.default: Import prepared!
  Prepared alicloud_security_group for import
alicloud_security_group.default: Refreshing state... [id=sg-bp11s5pka9pxtj6pn4xq]

Import successful!

The resources that were imported are shown above. These resources are now in
your Terraform state and will henceforth be managed by Terraform.
```

### 刷新资源
terraform refresh刷新当前state内容，调用云API拉取最新数据写入state文件
```shell
# terraform refresh
alicloud_vpc.vpc: Refreshing state... [id=vpc-bp1kulcyygsi727aay4hd]
alicloud_vswitch.vsw: Refreshing state... [id=vsw-bp1wgpgz9z8y2lfsl2beo]
alicloud_security_group.default: Refreshing state... [id=sg-bp11s5pka9pxtj6pn4xq]
alicloud_instance.wanzi_test: Refreshing state... [id=i-bp1gt9mb9asadff9r2zr]
```
