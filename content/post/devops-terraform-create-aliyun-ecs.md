---
title: "terraform自动化创建ECS"
date: 2021-02-26T17:22:42+08:00
lastmod: 2021-02-26T17:22:42+08:00
draft: false
description: "terraform自动化创建ECS"
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

## 快速创建一台阿里云ECS主机
### 指定terraform版本
这里我们指定了阿里云provider版本信息，并设置了terraform的版本要求
```shell
# mkdir aliyun-ecs-one && cd aliyun-ecs-one
# touch versions.tf
# vim versions.tf
 terraform {
  required_providers {
    alicloud = {
      source  = "aliyun/alicloud"
      version = "1.115.1"
    }
  }

  required_version = ">= 0.12"
}
```
### 配置变量

这里主要指定密钥对、云region、ECS账户和镜像信息
```shell
# vim variables.tf
# 阿里云子账户access_key
variable "alicloud_access_key" {
  default     = "LTAI4GBXXXXXXXXXXXXXXXXXXXXXX"
  description = "The Alicloud Access Key ID to launch resources. Support to environment 'ALICLOUD_ACCESS_KEY'."
}

# 阿里云子账户secret_key
variable "alicloud_secret_key" {
  default     = "4Z4gbl3dXXXXXXXXXXXXXXXXXXXXX"
  description = "The Alicloud Access Secret Key to launch resources.  Support to environment 'ALICLOUD_SECRET_KEY'."
}

# 阿里云区域，这里为杭州地区
variable "region" {
  default     = "cn-hangzhou"
  description = "The Alicloud region resources.  Support to environment 'REGION'."
}

# 设置阿里云杭州区域可用机房，这里设置为cn-hangzhou-i
variable "availability_zone" {
  description = "The available zone to launch ecs instance and other resources."
  default     = "cn-hangzhou-i"
}

# 设置镜像版本
variable "image_id" {
  default = "ubuntu_18_04_64_20G_alibase_20190624.vhd"
}

# 设置ECS实例类型，对于
variable "ecs_type" {
  default = "ecs.s6-c1m2.small"
}

# 指定ECS实例密码
variable "ecs_password" {
  default = "Test12345"
}

# 指定ECS实例磁盘类型，这里为普通云盘
variable "disk_category" {
  default = "cloud_efficiency"
}

# 设置磁盘大小
variable "disk_size" {
  default = "40"
}

# 设置上网扣费泪行，默认为PayByTraffic（按流量计费）
variable "internet_charge_type" {
  default = "PayByTraffic"
}

# 公共网络最大传出带宽，从1.7版本，默认设置大于0会自动申请独享公网IP地址
variable "internet_max_bandwidth_out" {
  default = 5
}
```

### 配置实例相关资源
这里由于是测试，创建实例需要提前创建vpc，vswitch，安全组，安全组规则
```shell
# vim main.tf
provider "alicloud" {
  region     = var.region
  access_key = var.alicloud_access_key
  secret_key = var.alicloud_secret_key
}

resource "alicloud_vpc" "vpc" {
  name       = "tf_test_foo"
  cidr_block = "10.100.0.0/16"
}

resource "alicloud_vswitch" "vsw" {
  vpc_id            = alicloud_vpc.vpc.id
  cidr_block        = "10.100.0.0/24"
  availability_zone = var.availability_zone
}

resource "alicloud_security_group" "default" {
  name   = "default"
  vpc_id = alicloud_vpc.vpc.id
}

resource "alicloud_security_group_rule" "allow_all_tcp" {
  type              = "ingress"
  ip_protocol       = "tcp"
  nic_type          = "intranet"
  policy            = "accept"
  port_range        = "1/65535"
  priority          = 1
  security_group_id = alicloud_security_group.default.id
  cidr_ip           = "0.0.0.0/0"
}

resource "alicloud_instance" "wanzi_test" {
  # cn-hangzhou
  availability_zone = var.availability_zone
  security_groups   = alicloud_security_group.default.*.id

  instance_type        = var.ecs_type
  system_disk_category = var.disk_category
  image_id             = var.image_id
  instance_name        = "wanzi_tf001"
  vswitch_id           = alicloud_vswitch.vsw.id
  password             = var.ecs_password
}
```

执行plan，模拟执行效果
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

创建云主机, 这个过程会请求阿里云API并在本地生成terraform state文件
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

通过以上操作，我们可以看出已经完成资源的创建，这个过程当前目录也会生成对应tfstate文件，这个数据非常重要，千万不要删除。
另外，后续可以通过terraform show查看我们创建资源数据信息。

## 批量创建多台ECS云主机
### 配置Module
由于https://registry.terraform.io/上已经有很多优秀的模块，我们这里直接拿来alibaba/ecs-instance/alicloud这个module进行操作即可。

更多关于官方ECS module参考这里：https://github.com/terraform-alicloud-modules/terraform-alicloud-ecs-instance

这里variables.tf和versions.tf配置还是基于第一步配置；这里我们在main.tf增加module资源批量创建ECS配置，如下：
```yaml
module "tf-instances" {
 source                      = "alibaba/ecs-instance/alicloud"
 region                      = "cn-hangzhou"
 number_of_instances         = "3"
 vswitch_id                  = alicloud_vswitch.vsw.id
 group_ids                   = [alicloud_security_group.default.id]
 private_ips                 = ["10.100.0.10", "10.100.0.11", "10.100.0.12"]
 image_ids                   = ["ubuntu_18_04_64_20G_alibase_20190624.vhd"]
 instance_type               = var.ecs_type
 internet_max_bandwidth_out  = 10
 associate_public_ip_address = true
 instance_name               = "my_module_instances_"
 host_name                   = "wanzi-cluster"
 internet_charge_type        = "PayByTraffic"
 password                    = var.ecs_password
 system_disk_category        = "cloud_ssd"
 data_disks = [
  {
    disk_category = "cloud_ssd"
    disk_name     = "my_module_disk"
    disk_size     = "50"
  }
 ]
}
```
这里需要注意默认情况下，internet_max_bandwidth_out配置以后，会自动申请一个独享公网IP地址，对于没有这个需求的，可以不用配置。

### 批量创建资源
```shell
➜ terraform apply
alicloud_vpc.vpc: Refreshing state... [id=vpc-bp1kulcyygsi727aay4hd]
alicloud_vswitch.vsw: Refreshing state... [id=vsw-bp1wgpgz9z8y2lfsl2beo]
alicloud_security_group.default: Refreshing state... [id=sg-bp11s5pka9pxtj6pn4xq]
alicloud_security_group_rule.allow_all_tcp: Refreshing state... [id=sg-bp11s5pka9pxtj6pn4xq:ingress:tcp:1/65535:intranet:0.0.0.0/0:accept:1]
alicloud_instance.wanzi_test: Refreshing state... [id=i-bp1gt9mb9asadff9r2zr]

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # module.tf-instances.alicloud_instance.this[0] will be created
  + resource "alicloud_instance" "this" {
      + availability_zone             = (known after apply)
      + credit_specification          = (known after apply)
      + deletion_protection           = false
      + description                   = "An ECS instance came from terraform-alicloud-modules/ecs-instance"
      + dry_run                       = false
      + host_name                     = "wanzi-cluster001"
      + id                            = (known after apply)
      + image_id                      = "ubuntu_18_04_64_20G_alibase_20190624.vhd"
      + instance_charge_type          = "PostPaid"
      + instance_name                 = "my_module_instances_001"
      + instance_type                 = "ecs.s6-c1m2.small"
      + internet_charge_type          = "PayByTraffic"
      + internet_max_bandwidth_in     = (known after apply)
      + internet_max_bandwidth_out    = 10
      + key_name                      = (known after apply)
      + password                      = (sensitive value)
      + private_ip                    = "10.100.0.10"
      + public_ip                     = (known after apply)
      + role_name                     = (known after apply)
      + security_enhancement_strategy = "Active"
      + security_groups               = [
          + "sg-bp11s5pka9pxtj6pn4xq",
        ]
      + spot_strategy                 = "NoSpot"
      + status                        = "Running"
      + subnet_id                     = (known after apply)
      + system_disk_category          = "cloud_ssd"
      + system_disk_performance_level = (known after apply)
      + system_disk_size              = 40
      + tags                          = {
          + "Name" = "my_module_instances_001"
        }
      + volume_tags                   = {
          + "Name" = "my_module_instances_001"
        }
      + vswitch_id                    = "vsw-bp1wgpgz9z8y2lfsl2beo"

      + data_disks {
          + category             = "cloud_efficiency"
          + delete_with_instance = true
          + encrypted            = false
          + name                 = "TF_ECS_Disk"
          + performance_level    = (known after apply)
          + size                 = 40
        }
    }

  # module.tf-instances.alicloud_instance.this[1] will be created
  + resource "alicloud_instance" "this" {
      + availability_zone             = (known after apply)
      + credit_specification          = (known after apply)
      + deletion_protection           = false
      + description                   = "An ECS instance came from terraform-alicloud-modules/ecs-instance"
      + dry_run                       = false
      + host_name                     = "wanzi-cluster002"
      + id                            = (known after apply)
      + image_id                      = "ubuntu_18_04_64_20G_alibase_20190624.vhd"
      + instance_charge_type          = "PostPaid"
      + instance_name                 = "my_module_instances_002"
      + instance_type                 = "ecs.s6-c1m2.small"
      + internet_charge_type          = "PayByTraffic"
      + internet_max_bandwidth_in     = (known after apply)
      + internet_max_bandwidth_out    = 10
      + key_name                      = (known after apply)
      + password                      = (sensitive value)
      + private_ip                    = "10.100.0.11"
      + public_ip                     = (known after apply)
      + role_name                     = (known after apply)
      + security_enhancement_strategy = "Active"
      + security_groups               = [
          + "sg-bp11s5pka9pxtj6pn4xq",
        ]
      + spot_strategy                 = "NoSpot"
      + status                        = "Running"
      + subnet_id                     = (known after apply)
      + system_disk_category          = "cloud_ssd"
      + system_disk_performance_level = (known after apply)
      + system_disk_size              = 40
      + tags                          = {
          + "Name" = "my_module_instances_002"
        }
      + volume_tags                   = {
          + "Name" = "my_module_instances_002"
        }
      + vswitch_id                    = "vsw-bp1wgpgz9z8y2lfsl2beo"

      + data_disks {
          + category             = "cloud_efficiency"
          + delete_with_instance = true
          + encrypted            = false
          + name                 = "TF_ECS_Disk"
          + performance_level    = (known after apply)
          + size                 = 40
        }
    }

  # module.tf-instances.alicloud_instance.this[2] will be created
  + resource "alicloud_instance" "this" {
      + availability_zone             = (known after apply)
      + credit_specification          = (known after apply)
      + deletion_protection           = false
      + description                   = "An ECS instance came from terraform-alicloud-modules/ecs-instance"
      + dry_run                       = false
      + host_name                     = "wanzi-cluster003"
      + id                            = (known after apply)
      + image_id                      = "ubuntu_18_04_64_20G_alibase_20190624.vhd"
      + instance_charge_type          = "PostPaid"
      + instance_name                 = "my_module_instances_003"
      + instance_type                 = "ecs.s6-c1m2.small"
      + internet_charge_type          = "PayByTraffic"
      + internet_max_bandwidth_in     = (known after apply)
      + internet_max_bandwidth_out    = 10
      + key_name                      = (known after apply)
      + password                      = (sensitive value)
      + private_ip                    = "10.100.0.12"
      + public_ip                     = (known after apply)
      + role_name                     = (known after apply)
      + security_enhancement_strategy = "Active"
      + security_groups               = [
          + "sg-bp11s5pka9pxtj6pn4xq",
        ]
      + spot_strategy                 = "NoSpot"
      + status                        = "Running"
      + subnet_id                     = (known after apply)
      + system_disk_category          = "cloud_ssd"
      + system_disk_performance_level = (known after apply)
      + system_disk_size              = 40
      + tags                          = {
          + "Name" = "my_module_instances_003"
        }
      + volume_tags                   = {
          + "Name" = "my_module_instances_003"
        }
      + vswitch_id                    = "vsw-bp1wgpgz9z8y2lfsl2beo"

      + data_disks {
          + category             = "cloud_efficiency"
          + delete_with_instance = true
          + encrypted            = false
          + name                 = "TF_ECS_Disk"
          + performance_level    = (known after apply)
          + size                 = 40
        }
    }

Plan: 3 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

module.tf-instances.alicloud_instance.this[2]: Creating...
module.tf-instances.alicloud_instance.this[1]: Creating...
module.tf-instances.alicloud_instance.this[0]: Creating...
module.tf-instances.alicloud_instance.this[1]: Still creating... [10s elapsed]
module.tf-instances.alicloud_instance.this[2]: Still creating... [10s elapsed]
module.tf-instances.alicloud_instance.this[0]: Still creating... [10s elapsed]
module.tf-instances.alicloud_instance.this[1]: Still creating... [20s elapsed]
module.tf-instances.alicloud_instance.this[0]: Still creating... [20s elapsed]
module.tf-instances.alicloud_instance.this[2]: Still creating... [20s elapsed]
module.tf-instances.alicloud_instance.this[0]: Creation complete after 21s [id=i-bp1hwbo4htk8sbwxtk6o]
module.tf-instances.alicloud_instance.this[1]: Creation complete after 21s [id=i-bp17lh41gywyih0xg6we]
module.tf-instances.alicloud_instance.this[2]: Creation complete after 22s [id=i-bp11zlrl6vxeaerz4ad0]

Apply complete! Resources: 3 added, 0 changed, 0 destroyed.
```

至此，整个创建多ECS实例的操作已经完成，后续如果对当前已经部署ecs资源有调整，进行基本write/plan/apply操作即可，这个过程会重启阿里云实例。

