# DevOps相关内容整理

> ~~没有人比我更懂devops~~

## 概览

- [Ansible](https://www.ansible.com/)
- [Docker(19.03.12 CE)](https://www.docker.com/)
- [GitLab(CE)](https://about.gitlab.com/)
- [Harhor](https://goharbor.io/)
- [Istio](https://istio.io)/[OSM](https://openservicemesh.io/)
- [Jenkins](http://www.jenkins.io/)
- [Kubernetes(1.19.0)](https://kubernetes.io/)/[TKEStack](https://www.openatom.org/#/projectDetail/d7417b183f064dd2a7662762cffdc62d)
- [Maven](https://maven.apache.org/)
- [Nexus(OSS)](https://www.sonatype.com/)
- [Nightwatch](https://nightwatchjs.org/)
- [Prometheus](https://prometheus.io/)/Datadog/Influx/Graphite/New Relic/Wavefront
- [Sinopia](https://github.com/rlidwka/sinopia)
- [Webpack](https://webpack.js.org/)

> 原则上，优先采用开源软件，尽量避免采用闭源、商业软件（目前已知Docker的商业版受美国EAR限制）；优先寻找闭源商业软件的可替代方案

> TODO
>
> 1. ELK，k8s还是Istio记得自带日志收集分析功能，要确认
> 1. 还缺一个接收、分析、展示前端错误的功能节点

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

   I. 安装预处理（可选）

   - 系统要求

      - 操作系统采用CentOS 7，因为目前Docker仅提供CentOS 7 相关Package

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
            > wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
            > ```

      - 推荐采用`overlay2`存储驱动（`storage driver`）

         可通过`docker info`确认Docker所使用的存储驱动

         > 存储驱动需要OS文件系统的支持。CentOS 7默认使用`xfs`文件系统，可支持Docker的`overlay2`驱动，但仅当`ftype`选项设为`1`时有效。
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

      > `19.03.12`为当前的稳定版。由于更新频繁，实际安装前，请先从[这里](https://hub.docker.com/_/docker)查阅最新的稳定版版本号。**不推荐**采用`sudo yum install docker-ce docker-ce-cli containerd.io`的方式进行安装

      至此，Docker已经安装完成并运行，接下来可移步安装后处理，配置docker命令的权限（可选）

   - 离线安装

      前往<https://download.docker.com/linux/centos/>并选择CentOS版本（7，目前有且仅有），然后浏览至`x86_64/stable/Packages/`，下载相应`.rpm`包，分别是：

      ``` text
      docker-ce-19.03.12-3.el7.x86_64.rpm
      docker-ce-cli-19.03.12-3.el7.x86_64.rpm
      containerd.io-1.2.13-3.2.el7.x86_64.rpm
      ```

      安装Docker引擎，把路径换成你下载的包的路径

      ``` bash
      sudo yum install /path/to/containerd.io-1.2.13-3.2.el7.x86_64.rpm
      sudo yum install /path/to/docker-ce-19.03.12-3.el7.x86_64.rpm
      sudo yum install /path/to/docker-ce-cli-19.03.12-3.el7.x86_64.rpm
      ```

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

      > 避免在执行上述操作前运行Docker CLI，会导致Docker CLI自动生成的`~/.docker/`路径（包括其中的路径和文件）的权限、所有者不匹配

   - 配置Docker Daemon

      ``` bash
      # 创建/etc/docker目录
      mkdir /etc/docker

      # 设置 daemon
      cat > /etc/docker/daemon.json <<EOF
      {
        "exec-opts": ["native.cgroupdriver=systemd"],
        "log-driver": "json-file",
        "log-opts": {
          "max-size": "100m"
        },
        "storage-driver": "overlay2",
        "storage-opts": [
          "overlay2.override_kernel_check=true"
        ]
      }
      EOF
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

1. 构建基础镜像

   - Java（Alpine + glibc + openjdk）

      - 方案1（在线 + 自定义）
         1. 构建Java基础镜像（Alpine + glibc），Dockerfile如下：

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

         1. 构建JRE镜像（Alpine + glibc + dragonwell）

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

         1. 上传镜像

            > TODO 看完Harbor后写

      - 方案2：（离线 + 自定义）

         > TODO

1. 构建应用镜像

   - 前端

      > TODO vue/react + nginx

   - 后端

      spring-boot应用

      ``` dockerfile
      # 指定基础镜像，这是分阶段构建的前期阶段
      FROM path/to/harbor/dragonwell:8.4.4_alpine_3.12.0_glibc_2.32-r0 as builder
      # 执行工作目录
      WORKDIR application
      # 配置参数
      ARG JAR_FILE=target/*.jar
      # 将编译构建得到的jar文件复制到镜像空间中
      COPY ${JAR_FILE} application.jar
      # 通过工具spring-boot-jarmode-layertools从application.jar中提取拆分后的构建结果
      RUN java -Djarmode=layertools -jar application.jar extract

      # 正式构建镜像
      FROM path/to/harbor/dragonwell:8.4.4_alpine_3.12.0_glibc_2.32-r0
      MAINTAINER usr <usr@mail.com>
      WORKDIR application
      # 前一阶段从jar中提取除了多个文件，这里分别执行COPY命令复制到镜像空间中，每次COPY都是一个layer
      COPY --from=builder application/dependencies/ ./
      COPY --from=builder application/spring-boot-loader/ ./
      COPY --from=builder application/snapshot-dependencies/ ./
      COPY --from=builder application/application/ ./
      ENTRYPOINT ["java", "org.springframework.boot.loader.JarLauncher"]
      ```

      > spring-boot版本须为2.3.0RELEASE及以上版本；此Dockerfile应位于Maven项目sub(1\~n)-implements内`pom.xml`所在目录。关于sub(1\~n)-implements的说明，请参见Maven一节

> TODO 写个shell->等看完Ansible的playbook再说？数据卷如何管理？->查一下<https://github.com/ClusterHQ/flocker>？MAINTAINER删了怎么样？

## Harbor

> TODO
> 镜像的管理？

## Webpack

## Maven

在每台后端（Java）开发机上部署，CI/CD的重要一环。因为IDE中基本都集成了Maven，所以安装之类的操作不在赘述，重点放在其他和DevOps相关的内容上

1. Maven Wrapper（mvnw）

   用以确保构建项目时，使用的Maven版本保持一致

   > `mvnw`的资料可查阅[这里](https://github.com/takari/maven-wrapper)。另外，除了保持版本统一外，其实`mvnw`还能玩一些别的花样，在[springfield](https://github.com/zennyx/springfield)里能看到用例

   - 确认方法

      如果当前项目支持mvnw，则项目根目录下应存在以下文件和文件夹：

      ``` text
      ├ .mvn
      |   └ wrapper
      |        ├ MavenWrapperDownloader.java
      |        ├ maven-wrapper.jar
      |        └ maven-wrapper.properties
      ├ mvnw
      └ mvnw.cmd
      ```

   - 获取mvnw支持

      - 使用[Spring Initializr](https://start.spring.io/)或者[STS](https://spring.io/tools)创建的spring项目，原生支持mvnw
      - 使用Maven 3.7.0及以上版本创建的项目，原生支持mvnw
      - 对于既存项目，也可以通过以下方式获得支持：

         直接执行Maven命令（在线）

         ``` bash
         cd path/to/yourmavenproject
         mvn -N io.takari:maven:0.7.7:wrapper -Dmaven=3.5.4
         ```

         > 0.7.7为wrapper-maven-plugin的版本，3.5.4表示期望Maven版本为3.5.4

         从上述原生支持的项目中拷贝相关文件至既存项目中（离线）

         > 注意离线的场合需要修改`maven-wrapper.properties`文件中的内容，以使用本地Maven，例如：

         ``` properties
         #distributionUrl=https://repo1.maven.org/maven2/org/apache/maven/apache-maven/3.5.4/apache-maven-3.5.4-bin.zip
         distributionUrl=file:///D:/java/apache-maven-3.5.4-bin.zip
         ```

         > TODO Jenkins或者GitLab那台主机估计要装Maven，看谁最终执行编译任务，要确认

   - 使用（以`clean install`为例）

      原先

      ``` bash
      mvn clean install
      ```

      获得`mvnw`支持后

      ``` bash
      ./mvnw clean install
      ```

      或者（在Win操作系统上）

      ``` shell
      mvnw.cmd clean install
      ```

   > TODO 确认下哪些模块需要mvnw支持，仅主项目还是子项目也需要

1. Maven项目结构

   ``` text
   parent
     └ main
         ├ sub1
         |    ├ sub1-interfaces
         |    └ sub1-implements
         ├ sub2
         |    ├ sub2-interfaces
         |    └ sub2-implements
         ...
         └ subn
              ├ subn-interfaces
              └ subn-implements
   ```

   parent（pom项目）：用以显式项目依赖/插件的版本，可使用的镜像及仓库

   main（pom项目）：用以管理下属各子模块项目的构建，指定`profiles`。main项目不一定要作为parent的子模块，平级亦可，只要main的pom以parent为父pom即可，这样parent还能为其他项目提供“模板”，增加灵活性

   sub(1\~n)（pom项目）：业务模块，可根据领域模型切分，更细致得管理本模块的构建，也便于分工

   sub(1\~n)-interfaces（jar项目）：所有接口、抽象及顶层POJO、异常放在interfaces内，最大限度减少外部调用时所需的依赖，降低出现“依赖地狱”的风险

   sub(1\~n)-implements（jar项目）：和sub(1\~n)-interfaces对应，所有的业务实现放在implements内，同时还须提供对容器化的支持，其`pom.xml`示例如下所示（spring-boot版本须2.3.0.RELEASE及以上）：

   ``` xml
   <?xml version="1.0" encoding="UTF-8"?>
   <project xmlns="http://maven.apache.org/POM/4.0.0"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
     <modelVersion>4.0.0</modelVersion>

     <!-- 引入wavefront需要额外配置依赖，应配置于parent项目中 -->
     <!--
     <properties>
       <wavefront.version>2.0.0</wavefront.version>
     </properties>
     -->

     <dependencies>
       <!-- 对外提供内建的终端（诸如指标、审计信息等），以便监控、管理应用 -->
       <dependency>
         <groupId>org.springframework.boot</groupId>
         <artifactId>spring-boot-starter-actuator</artifactId>
       </dependency>
       <!-- 指标监控类，按需选择使用 -->
       <dependency>
         <groupId>com.wavefront</groupId>
         <artifactId>wavefront-spring-boot-starter</artifactId>
       </dependency>
       <dependency>
         <groupId>io.micrometer</groupId>
         <artifactId>micrometer-registry-datadog</artifactId>
         <scope>runtime</scope>
       </dependency>
       <dependency>
         <groupId>io.micrometer</groupId>
         <artifactId>micrometer-registry-graphite</artifactId>
         <scope>runtime</scope>
       </dependency>
       <dependency>
         <groupId>io.micrometer</groupId>
         <artifactId>micrometer-registry-influx</artifactId>
         <scope>runtime</scope>
       </dependency>
       <dependency>
         <groupId>io.micrometer</groupId>
         <artifactId>micrometer-registry-new-relic</artifactId>
         <scope>runtime</scope>
       </dependency>
       <dependency>
         <groupId>io.micrometer</groupId>
         <artifactId>micrometer-registry-prometheus</artifactId>
         <scope>runtime</scope>
       </dependency>
     </dependencies>

     <!-- 引入wavefront需要额外配置依赖，应配置于parent项目中 -->
     <!--
     <dependencyManagement>
       <dependencies>
         <dependency>
           <groupId>com.wavefront</groupId>
           <artifactId>wavefront-spring-boot-bom</artifactId>
           <version>${wavefront.version}</version>
           <type>pom</type>
           <scope>import</scope>
         </dependency>
       </dependencies>
      </dependencyManagement>
      -->

     <build>
       <plugins>
         <plugin>
           <groupId>org.springframework.boot</groupId>
           <artifactId>spring-boot-maven-plugin</artifactId>
           <configuration>
             <layers>
               <!-- 用于生成分层文件layers.idx，添加依赖spring-boot-jarmode-layertools -->
               <enabled>true</enabled>
             </layers>
           </configuration>
         </plugin>
         <plugin>
           <groupId>com.spotify</groupId>
           <artifactId>dockerfile-maven-plugin</artifactId>
           <executions>
             <execution>
               <id>default</id>
               <goals>
                 <!-- 用于构建并上传镜像 -->
                 <!-- Dockerfile应位于pom.xml所在目录，即根目录 -->
                 <goal>build</goal>
                 <goal>push</goal>
               </goals>
             </execution>
           </executions>
           <configuration>
             <!-- 出于安全考虑，不在pom中直接配置私有仓库的安全凭证，而在调用插件实施构建时作为参数传入 -->
             <!-- mvn goal -Ddockerfile.username=... -Ddockerfile.password=... -->
             <repository>${docker.image.prefix}/${project.artifactId}</repository>
             <tag>${project.version}</tag>
           </configuration>
         </plugin>
       </plugins>
     </build>
   </project>
   ```

   > 参考资料：[打包](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-container-images)、[部署](https://docs.spring.io/spring-boot/docs/current/reference/html/deployment.html#deployment)、[插件](https://docs.spring.io/spring-boot/docs/current/maven-plugin/reference/html/#repackage)、[其他](https://github.com/spotify/dockerfile-maven/blob/master/docs/usage.md)

1. 课题

   - 用parent指定依赖版本对于短时间的开发没有问题，但长期运维的项目必定会碰到依赖升级的问题，这个时候如何处理比较好？

   - 各子模块的版本号该怎么管理？各子模块版本统一虽然方便管理，但会出现部分模块无更改提版、一次改动全部部署；各自管理则在相互依赖时会出现搞不清楚依赖哪个版本？-> 子模块间的相互依赖如何管理？

> TODO Jenkins pipeline

## GitLab(CE)

## Sinopia

## Nexus(OSS)

## Jenkins

## Kubernetes

至少3台主机或虚拟机用作主节点（Master Node），至少3台主机或虚拟机用作从节点（Work Node）

1. 安装

   使用`kubeadm`创建集群

      I. 安装预处理（可选）

      - 系统要求
         - 操作系统采用CentOS 7，保持统一
         - 每台主机/虚拟机2GB或更多的RAM（如果少于这个数值将会影响应用的运行内存）
         - 2CPU核或更多
         - 集群中的所有机器的网络彼此均能相互连接(公网和内网均可)
         - 确保每个节点上MAC地址和product_uuid的唯一性

            一般来讲，硬件设备会拥有唯一的地址，但是有些虚拟机的地址可能会重复。Kubernetes 使用这些值来唯一确定集群中的节点。 如果这些值在每个节点上不唯一，可能会导致安装失败
            - 可以使用命令`ip link`或`ifconfig -a`来获取网络接口的MAC地址
            - 可以使用`sudo cat /sys/class/dmi/id/product_uuid`命令对product_uuid 校验

         - 确保开启主机/虚拟机上的以下端口

            查看端口状况

            ``` bash
            netstat -anp
            ```

            打开端口（替换`portNumber`为所需打开端口号）

            ``` bash
            iptables -A INPUT -p tcp --dport portNumber -j ACCEPT
            ```

            关闭端口（替换`portNumber`为所需关闭端口号）

            ``` bash
            iptables -A OUTPUT -p tcp --dport portNumber -j DROP
            ```

            保存设置

            ``` bash
            service iptables save
            ```

            主节点用

            协议|方向|端口范围|作用|使用者
            :--:|:--:|--|--|--
            TCP|入站|6443|Kubernetes API Server|所有组件
            TCP|入站|2379-2380|etcd server client API|kube-apiserver、etcd
            TCP|入站|10250|Kubelet API|kubelet自身、Control plane组件
            TCP|入站|10251|kube-scheduler|kube-scheduler自身
            TCP|入站|10252|kube-controller-manager|kube-controller-manager自身

            从节点用

            协议|方向|端口范围|作用|使用者
            :--:|:--:|--|--|--
            TCP|入站|10250|Kubelet API|kubelet自身、Control plane组件
            TCP|入站|30000-32767|NodePort服务|所有组件

         - **禁用**`swap`（交换分区），以保证`kubelet`工作正常

           查看/etc/fstab

           ``` bash
           more /etc/fstab
           ```

           找到swap分区的记录，把加载swap分区的那行记录注释掉，例如：

           ``` text
           #/dev/mapper/cl-swap     swap                    swap    defaults        0 0
           ```

           重启主机/虚拟机

           ``` bash
           systemctl reboot
           ```

           使用`free -m`命令确认，显示结果为应为：

           ``` bash
                         total        used        free      shared  buff/cache   available
           Swap:             0           0           0
           ```

      - 检查网络适配器

         如果存在一个以上的网络适配器，同时Kubernetes组件通过默认路由不可达的场合，建议预先添加 IP 路由规则，以便Kubernetes集群可以通过对应的适配器完成连接

      - 确保`iptables`工具能够处理bridged traffic

         执行`lsmod | grep br_netfilter`命令，确保`br_netfilter`模块已加载；或者通过`sudo modprobe br_netfilter`命令显式加载

         然后，确保`sysctl`的配置文件中，`net.bridge.bridge-nf-call-iptables`被设置为`1`，例如：

         ``` bash
         cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
         net.bridge.bridge-nf-call-ip6tables = 1
         net.bridge.bridge-nf-call-iptables = 1
         EOF

         sudo sysctl --system
         ```

         > 这里官方文档的中文版本和英/日版的说明不一致，暂时以英/日版为准

      II. 安装方式

      - 在线安装

         安装CRI-O（容器运行时），如果安装Kubernetes的主机/虚拟机已经安装了Docker，此步骤可忽略

         ``` bash
         modprobe overlay
         modprobe br_netfilter

         # 设置必需的sysctl参数，这些参数在重新启动后仍然存在
         cat > /etc/sysctl.d/99-kubernetes-cri.conf <<EOF
         net.bridge.bridge-nf-call-iptables  = 1
         net.ipv4.ip_forward                 = 1
         net.bridge.bridge-nf-call-ip6tables = 1
         EOF

         sysctl --system

         # 设置环境变量
         OS=CentOS_7
         VERSION=1.19:1.19.0

         curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable.repo https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/devel:kubic:libcontainers:stable.repo
         curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.repo https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$VERSION/$OS/devel:kubic:libcontainers:stable:cri-o:$VERSION.repo

         yum install cri-o

         # 启动CRI-O
         systemctl daemon-reload
         systemctl start crio
         ```

         > **注意：** CRI-O的主、次版本必须和Kubernetes的主、次版本保持一致

         安装kubeadm、kubelet和kubectl

         需要在每台机器上安装以下的软件包：
         - kubeadm：用来初始化集群的指令
         - kubelet：在集群中的每个节点上用来启动pod和容器等
         - kubectl：用来与集群通信的命令行工具

         ``` bash
         cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
         [kubernetes]
         name=Kubernetes
         baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
         enabled=1
         gpgcheck=1
         repo_gpgcheck=1
         gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
         exclude=kubelet kubeadm kubectl
         EOF

         # 将SELinux设置为permissive模式（相当于将其禁用）
         sudo setenforce 0
         sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

         sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

         sudo systemctl enable --now kubelet
         ```

         > **注意：**
         > 通过运行命令`setenforce 0`和`sed ...`将SELinux设置为permissive模式可以有效的将其禁用。 这是允许容器访问主机文件系统所必须的，例如正常使用pod网络。 目前不得不这么做，直到kubelet升级支持SELinux为止

         > TODO: 这几个repo的地址肯定连不上……需要找替代的

         kubelet现在每隔几秒就会重启，陷入了一个等待kubeadm发布指令的死循环。这是正常现象， 初始化控制平面后，kubelet 将正常运行

         在control-plane node（控制平面节点）上配置kubelet使用的cgroup驱动程序。须将`cgroupDriver`传递`kubeadm init`，如下：

         ``` yml
         apiVersion: kubelet.config.k8s.io/v1beta1
         kind: KubeletConfiguration
         cgroupDriver: <value>
         ```

         > **注意：** **仅当**cgroup驱动程序不是cgroupfs时才需要这么做，因为这就是kubelet中的默认值

         重启kubelet

         ``` bash
         systemctl daemon-reload
         systemctl restart kubelet
         ```

         安装Pod网络安全附件

         > TODO

         接着使用`kubeadm`创建集群

         初始化control-plane node（控制平面节点）

         > 控制平面节点是运行控制平面组件的主机/虚拟机， 包括`etcd`（集群数据库） 和`API Server`（命令行工具`kubectl`与之通信）。

         1. （推荐）如果打算把单控制平面`kubeadm`集群升级到高可用，必须设置`--control-plane-endpoint`为所有控制平面节点指定共享终端。该终端可以是负载均衡器的DNS名称或IP地址
         1. 选择一个Pod网络附件（network add-on），并确认其是否需要将任何参数传递给`kubeadm init`。根据选定的第三方驱动，可能需要通过设置参数`--pod-network-cidr`指定驱动
         1. （可选）从版本1.14开始，`kubeadm`尝试使用一系列众所周知的域套接字路径来检测Linux上的容器运行时。要使用不同的容器运行时，又或者在预配置的节点上安装了多个容器，请为`kubeadm init`指定`--cri-socket`参数
         1. （可选）除非另有说明，否则`kubeadm`使用与默认网关关联的网络接口来设置此控制平面节点API server的广播地址。 要使用其他网络接口，请为`kubeadm init`设置 `--apiserver-advertise-address=<ip-address>`参数。 要部署使用IPv6地址的Kubernetes集群， 必须指定一个IPv6地址，例如`--apiserver-advertise-address=fd00::101`
         1. （可选）在`kubeadm init`之前运行`kubeadm config images pull`，以验证与gcr.io容器镜像仓库的连通性

         要初始化控制平面节点，请运行：

         ``` bash
         kubeadm init <args>
         ```

      - 离线安装

         安装Containerd（容器运行时），如果安装Kubernetes的主机/虚拟机已经安装了Docker，此步骤可忽略

         > 通过添加repo再安装CRI-O的方式离线估计不适用，考虑用Containerd代替

         ``` bash
         cat > /etc/modules-load.d/containerd.conf <<EOF
         overlay
         br_netfilter
         EOF

         modprobe overlay
         modprobe br_netfilter

         # 设置必需的sysctl参数，这些参数在重新启动后仍然存在
         cat > /etc/sysctl.d/99-kubernetes-cri.conf <<EOF
         net.bridge.bridge-nf-call-iptables  = 1
         net.ipv4.ip_forward                 = 1
         net.bridge.bridge-nf-call-ip6tables = 1
         EOF

         sysctl --system

         # 安装（假设containerd已经被下载并存放在/path/to目录下）
         yum install /path/to/containerd.io-1.2.13-3.2.el7.x86_64.rpm

         # 配置
         mkdir -p /etc/containerd
         containerd config default > /etc/containerd/config.toml

         # 重启
         systemctl restart containerd
         ```

         安装kubeadm、kubelet和kubectl

         > 官方未提供，暂时咕了╮(╯▽╰)╭

      III. 安装后处理（可选）

   使用`kops`安装Kubernetes

      不考虑

   使用`Kubespray`安装Kubernetes

      不考虑

1. 更新

> TODO
>
> 1. 高可用
> 1. 节点版本更新（不是Deployment，是指kube-apiserver、kubelet等的更新）
> 1. [Kubernetes Probes with spring-boot](https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-features.html#production-ready-kubernetes-probes)
> 1. 调查整理ConfigMap、Ingress等Spring Cloud的替代方案（不全，一部分需由Istio或OSM替代）
> 1. 各节点内的应用之间如何通信？能使用哪些协议？

## Istio

## Nightwatch

## Prometheus

> TODO 监测缺一个东西：调用链，或者说调用依赖的监控，记得k8s有，要确认；可能还要加一个：Flyway

> TODO 杂项：
>
> 1. 发布的版本管理 -> <http://semver.org/>
