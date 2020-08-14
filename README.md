# DevOps相关内容整理

> ~~没有人比我更懂devops~~

## 概览

- [Ansible](https://www.ansible.com/)
- [Docker(CE)](https://www.docker.com/)
- [GitLab(CE)](https://about.gitlab.com/)
- [Harhor](https://goharbor.io/)
- [Istio](https://istio.io)/[OSM](https://openservicemesh.io/)
- [Jenkins](http://www.jenkins.io/)
- [Kubernetes](https://kubernetes.io/)
- [Maven](https://maven.apache.org/)
- [Nexus(OSS)](https://www.sonatype.com/)
- [Nightwatch](https://nightwatchjs.org/)
- [Prometheus](https://prometheus.io/)
- [Sinopia](https://github.com/rlidwka/sinopia)
- [Webpack](https://webpack.js.org/)

> 原则上，优先采用开源软件，尽量避免采用闭源商业软件；优先寻找闭源商业软件的可替代方案

> TODO ELK，k8s还是Istio记得自带日志收集分析功能，要确认

``` sequence
participant IDE
participant GitLab
participant Jenkins
participant Webpack
participant Sinopia
participant Maven
participant Nexus
participant Docker
participant Harbor
participant Kubernetes
participant Nightwatch
participant Prometheus

IDE->GitLab: 提交源代码
GitLab->Jenkins: WebHook
Jenkins->Webpack: 调用
Webpack->Webpack: 转译
Webpack->Webpack: 自动化单元测试
Webpack-->IDE: 单元测试未通过
Webpack->Sinopia: 打包并上传至仓库
Jenkins->Maven: 调用
Maven->Maven: 编译
Maven->Maven: 自动化单元测试
Maven-->IDE: 单元测试未通过
Maven->Nexus: 打包并上传至仓库
Jenkins->Docker: 调用
Docker->Docker: 生成镜像
Docker->Harbor: 上传镜像
Jenkins->Kubernetes: 部署
Jenkins->Nightwatch: 自动化端对端测试
Nightwatch-->IDE: 端对端测试未通过
Prometheus->>Kubernetes: 监控
```

## Ansible

## Docker(CE)

在至少一台主机或虚拟机上部署Docker，用于制作镜像文件。

1. 安装

   > 操作系统采用CentOS 8，之后如无说明，一律默认采用CentOS 8

   I. 安装预处理

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

   II. 安装方式

   - 在线安装

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

      > `19.03.12`为当前的稳定版。由于更新频繁，实际安装前，请先从[这里](https://hub.docker.com/_/docker)查阅最新的稳定版版本号。**严禁**采用`sudo yum install docker-ce docker-ce-cli containerd.io`的方式进行安装

      至此，Docker已经安装完成并运行，接下来可移步安装后处理，配置docker命令的权限（可选）

   - 离线安装

      > TODO 待更新

   - 脚本安装（需在线，仅适用于测试及开发场合）

      不考虑

   III. 安装后处理（可选）

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

   - 安全性管理

      > TODO SELinux等等

1. 建立基础镜像

   - Java（Alpine + glibc + openjdk）

      - 方案1（在线 + 自定义）
         1. 建立Java基础镜像（Alpine + glibc），Dockerfile如下：

            ``` dockerfile
            FROM alpine:3.12.0
            MAINTAINER usr <usr@mail.com>
            RUN apk add --no-cache ca-certificates curl openssl binutils xz tzdata \
              && ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
              && echo "Asia/Shanghai" > /etc/timezone \
              && GLIBC_VER="2.32-r0" \
              && ALPINE_GLIBC_REPO="https://github.com/sgerrand/alpine-pkg-glibc/releases/download" \
              && curl -Ls ${ALPINE_GLIBC_REPO}/${GLIBC_VER}/glibc-${GLIBC_VER}.apk > /tmp/${GLIBC_VER}.apk \
              && apk add --allow-untrusted /tmp/${GLIBC_VER}.apk \
              && curl -Ls https://www.archlinux.org/packages/core/x86_64/gcc-libs/download > /tmp/gcc-libs.tar.xz \
              && mkdir /tmp/gcc \
              && tar -xf /tmp/gcc-libs.tar.xz -C /tmp/gcc \
              && mv /tmp/gcc/usr/lib/libgcc* /tmp/gcc/usr/lib/libstdc++* /usr/glibc-compat/lib \
              && strip /usr/glibc-compat/lib/libgcc_s.so.* /usr/glibc-compat/lib/libstdc++.so* \
              && curl -Ls https://www.archlinux.org/packages/core/x86_64/zlib/download > /tmp/libz.tar.xz \
              && mkdir /tmp/libz \
              && tar -xf /tmp/libz.tar.xz -C /tmp/libz \
              && mv /tmp/libz/usr/lib/libz.so* /usr/glibc-compat/lib \
              && apk del binutils \
              && rm -rf /tmp/${GLIBC_VER}.apk /tmp/gcc /tmp/gcc-libs.tar.xz /tmp/libz /tmp/libz.tar.xz /var/cache/apk/*
            ```

            > 注意：1. 文中`usr`和`usr@mail.com`需根据实际情况替换；2. 文中把时区设为了中国上海，可根据实际情况替换; 3. gcclib的下载链接没有版本号，下次制作镜像时下载的gcclib版本可能变动，但是提前下载后COPY/ADD会增加分层吧？再考虑一下

         1. 上传镜像

            > TODO 看完Harbor后写

         1. 建立JRE镜像（Alpine + glibc + dragonwell）

            ``` dockerfile
            FROM path/to/harbor/alpine:3.12.0_glibc_2.32-r0
            MAINTAINER usr <usr@mail.com>
            RUN DRAGONWELL_VER="8.4.4" \
               && DRAGONWELL_REPO="https://dragonwell.oss-cn-shanghai.aliyuncs.com/8" \
               && JAVA_VER="8u262-b10"
               && curl -Ls ${DRAGONWELL_REPO}/${DRAGONWELL_VER}-GA/Alibaba_Dragonwell_${DRAGONWELL_VER}-GA_Linux_x64.tar.gz > /tmp/dragonwell.tar.gz \
               && mkdir /tmp/dragonwell \
               && tar -xf /tmp/dragonwell.tar.gz -C /tmp/dragonwell \
               && mv /tmp/dragonwell/jdk${JAVA_VER}/jre /opt/java \
               && rm -rf /tmp/dragonwell.tar.gz /tmp/dragonwell
            RUN chmod +x /opt/java
            ENV JAVA_HOME=/opt/java
            ENV PATH="$JAVA_HOME/jre/bin:${PATH}"
            ```

            > 嫌麻烦也可以直接使用`dragonwell`官方提供的镜像，详见[Use Dragonwell8 docker image](https://github.com/alibaba/dragonwell8/wiki/Use-Dragonwell8-docker-image)和[Use Dragonwell 11 docker images](https://github.com/alibaba/dragonwell11/wiki/Use-Dragonwell-11-docker-images)，只是注意一点，这些是`jdk`

            > TODO 目录用/usr/local还是/opt？

      - 方案2：（离线 + 自定义）

> TODO 写个shell->等看完Ansible的playbook再说？数据卷如何管理？->查一下<https://github.com/ClusterHQ/flocker>？

## Harbor

> TODO
> 镜像的管理？

## Webpack

## Maven

## GitLab(CE)

## Sinopia

## Nexus(OSS)

## Jenkins

## Kubernetes

> TODO
> Master节点ha？

## Istio

## Nightwatch

## Prometheus
