# DevOps相关内容整理

> ~~没有人比我更懂devops~~

## 概览

- [Ansible](https://www.ansible.com/)
- [Docker(CE)](https://www.docker.com/)
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

> 原则上，优先采用开源软件，尽量避免采用闭源商业软件；优先寻找闭源商业软件的可替代方案

> 课题： ELK

## Ansible

## Docker(CE)

在至少一台主机或虚拟机上部署Docker，用于制作镜像文件。

1. 安装

   > 操作系统采用CentOS 8，之后如无说明，一律默认采用CentOS 8

   1. 前置条件

      - 系统要求

         - `centos-extras`仓库必须启用。该仓库默认启用，但如果被禁用，可通过以下方式重新启用：

            1. 打开`CentOS-Base.repo`

                ``` bash
                sudo vi /etc/yum.repos.d/CentOS-Base.repo
                ```

            1. 找到`[extra]`部分，将`enabled`从`0`改为`1`

                ``` text
                enabled=1
                ```

            1. 保存退出，执行以下命令确认

                ``` bash
                # 清楚缓存，并确保变更立刻生效
                sudo yum clean all
                # 重建缓存
                sudo yum makecache
                # 显示所有可用repo
                sudo yum list
                ```

                > 这种情况可能发生在变更了仓库源的时候，如：
                >
                > ``` bash
                > # 使用阿里的镜像源
                > wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-8.repo
                > ```

         - 推荐采用`overlay2`存储驱动（`storage driver`）

            可通过`docker info`确认Docker所使用的存储驱动

            > 存储驱动需要OS文件系统的支持。CentOS 8默认使用`xfs`文件系统，可支持Docker的`overlay2`驱动，但仅当`ftype`选项设为`1`时有效。
            > 可通过以下命令确认文件系统是否支持`overlay2`：
            >
            > ``` bash
            > xfs_info /mnt/partition # 假设/mnt/partition为用户挂载分区
            > ```
            >
            > 如果不满足需求（`ftype=0`）,须通过以下命令重新格式化分区：
            >
            > ```bash
            > sudo mkfs.xfs -n ftype=1 /mnt/partition # 假设/mnt/partition为用户挂载分区
            > ```
            >
            > 如果当前存储驱动不是`overlay2`，可按照[这里](https://docs.docker.com/storage/storagedriver/overlayfs-driver/)的说明进行切换

            如果当前存储驱动不是`overlay2`，可通过向Docker守护进程传递相应参数切换

      - 删除旧版本

         旧版本的Docker被称为`docker`或者`docker-engine`，可通过以下命令确认卸载：

         ``` bash
         sudo yum remove docker \
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
         # 安装yum-utils包，该包提供yum-config-manager
         sudo yum install -y yum-utils
         # 通过yum-config-manager设置stable repository
         sudo yum-config-manager \
               --add-repo \
               https://download.docker.com/linux/centos/docker-ce.repo
         # 安装docker、CLI以及依赖
         sudo yum install docker-ce-19.03.12 docker-ce-cli-19.03.12 containerd.io
         ```

         > 如果提示接收GPG key，验证其指纹是否匹配`060A 61C5 1B55 8A7F 742B 77AA C52F EB6B 621E 9F35`，如果是，接收

         验证Docker安装是否正确：

         ``` bash
         # 启动Docker
         sudo systemctl start docker
         # 通过运行hello-world进行验证Docker是否安装成功
         sudo docker run hello-world
         ```

         > `19.03.12`为当前的稳定版。由于更新频繁，实际安装前，请先从[这里](https://hub.docker.com/_/docker)查阅最新的稳定版版本号。***严禁***采用`sudo yum install docker-ce docker-ce-cli containerd.io`的方式进行安装

         至此，Docker已经安装完成并运行，接下来可移步安装后处理，配置docker命令的权限（可选）

      1. 离线安装

      1. 脚本安装（需在线，仅适用于测试及开发场合）

         不考虑

      1. 安装后处理

         本节内容可选，但出于[便利的考虑](https://docs.docker.com/engine/install/linux-postinstall/)推荐执行：

         - 用户管理

           执行以下操作可在执行`docker`命令时省略`sudo`：

            ``` bash
            # 创建docker组
            sudo groupadd docker
            # 添加你的用户到docker组
            sudo usermod -aG docker $USER
            # 登出再登入，或者执行以下命令，使变更生效
            newgrp docker
            # 验证
            docker run hello-world
            ```

         - 服务管理

            执行以下操作，使Docker随系统自启动：

            ``` bash
            sudo systemctl enable docker
            ```

            > 把`enable`改成`disable`就是反向操作

         更多后处理，请参阅[这里](https://docs.docker.com/engine/install/linux-postinstall/)

1. 建立基础镜像

   1. Java

   1. Node.js

   1. 其他

> 课题： 写个shell？数据卷如何管理（https://github.com/ClusterHQ/flocker查一下）？

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
