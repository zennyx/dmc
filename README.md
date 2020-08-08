# DevOps相关内容整理

> ~~没有人比我更懂devops~~

## 概览

- [Ansible](https://www.ansible.com/)
- [Docker](https://www.docker.com/)
- [GitLab(CE)](https://about.gitlab.com/)
- [Harhor](https://goharbor.io/)
- [Istio](https://istio.io)
- [Jenkins](http://www.jenkins.io/)
- [Kubernetes](https://kubernetes.io/)
- [Maven](https://maven.apache.org/)
- [Nexus(OSS)](https://www.sonatype.com/)
- [Nightwatch](https://nightwatchjs.org/)
- [Prometheus](https://prometheus.io/)
- [Sinopia](https://github.com/rlidwka/sinopia)
- [Webpack](https://webpack.js.org/)

> 原则上，优先采用开源软件，尽量避免采用闭源商业软件，如无法避免，须有可替代方案

## Ansible

## Docker

1. 安装

   > 操作系统采用CentOS 8，之后如无说明，一律默认采用CentOS 8

   1. 前提

      1. 系统要求

         1. `centos-extras`仓库必须启用。该仓库默认启用，但如果被禁用，可通过以下方式重新启用：

            1. 打开`CentOS-Base.repo`

                ``` bash
                vi /etc/yum.repos.d/CentOS-Base.repo
                ```

            1. 找到`[extra]`部分，将`enabled`从`0`改为`1`

                ``` text
                enabled=1
                ```

            1. 保存退出，执行以下命令确认

                ``` bash
                yum clean all # 清楚缓存，并确保变更立刻生效
                yum makecache # 重建缓存
                yum list # 显示所有可用repo
                ```

                > 这种情况可能发生在变更了仓库源的时候，如：
                >
                > ``` bash
                > wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-8.repo # 使用阿里的镜像源
                > ```

         1. Docker推荐采用`overlay2`存储驱动（`storage driver`）

            可通过`docker info`确认Docker所使用的存储驱动

            > 存储驱动需要OS文件系统的支持。CentOS 8默认使用`xfs`文件系统，可支持`Docker`的`overlay2`驱动，但仅当`ftype`选项设为`1`时有效。
            > 可通过以下命令确认文件系统是否支持`overlay2`：
            >
            > ``` bash
            > xfs_info /dev/sda # 假设/dev/sda为用户挂载分区
            > ```
            >
            > 如果不满足需求（`ftype=0`）,须通过以下命令重新格式化分区：
            >
            > ```bash
            > mkfs.xfs -n ftype=1 /dev/sda # 假设/dev/sda为用户挂载分区
            > ```
            >
            > 更多信息，可查阅[这里](https://docs.docker.com/storage/storagedriver/overlayfs-driver/)

      1. 删除旧版本

         旧版本的Docker被称为`docker`或者`docker-engine`，可通过以下命令确认卸载：

         ``` bash
         yum remove docker \
                    docker-client \
                    docker-client-latest \
                    docker-common \
                    docker-latest \
                    docker-latest-logrotate \
                    docker-logrotate \
                    docker-engine
         ```

   1. 安装方式

      1. 在线安装

         执行以下命令进行安装：

         ``` bash
         yum install -y yum-utils # 安装yum-utils包，该包提供yum-config-manager
         yum-config-manager \
                 --add-repo \
                 https://download.docker.com/linux/centos/docker-ce.repo # 通过yum-config-manager设置stable repository
         yum install docker-ce docker-ce-cli containerd.io # 安装docker、CLI以及依赖
         ```

         > 如果提示接收GPG key，验证其指纹是否匹配`060A 61C5 1B55 8A7F 742B 77AA C52F EB6B 621E 9F35`，如果是，接收

         <font color='red'>***这里安装没有指定包的版本，要确认***</font>

      1. 离线安装

      1. 脚本安装（需在线，仅适用于测试及开发场合）

1. 建立基础镜像

   1. Java

   1. Node.js

   1. 其他

## Harbor

> 课题：
> 镜像的管理？

## Webpack

## Maven

## GitLab(CE)

## Sinopia

## Nexus(OSS)

## Jenkins

## Kubernetes

> 课题：
> Master节点ha？

## Istio

## Nightwatch

## Prometheus
