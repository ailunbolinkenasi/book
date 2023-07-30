Office Dockerfile：[官方例子](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
Docker可以通过从Dockerfile 包含所有命令的文本文件中读取指令自动构建镜像，以便构建给定镜像。

Dockerfiles 使用特定的格式并使用一组特定的指令。您可以在Dockerfile Reference页面上了解基础知识 。如果你是新手写作Dockerfile，你应该从那里开始。

本文档介绍了由 Docker，Inc. 和 Docker 社区推荐的用于构建高效镜像的最佳实践和方法。要查看许多实践和建议，请查看Dockerfile for buildpack-deps。

## 一些建议
一些我经常使用的时候会存在的问题
### Context
当你发出一个`docker build`命令时，当前的工作目录被称为构建上下文。默认情况下，Dockerfile 就位于该路径下，当然您也可以`-f`参数来指定不同的位置。
例如: 指定`Dockerfile`在当前目录
```shell
docker build -f Dockerfile  . -t test:v1
```
> 无论 Dockerfile 在什么地方，当前目录中的所有文件内容都将作为构建上下文发送到 Docker 守护进程中去。

在构建的时候包含不需要的文件会导致更大的构建上下文和更大的镜像大小。这会增加构建时间，拉取和推送镜像的时间以及容器的运行时间大小。要查看您的构建环境有多大，请在构建您的系统时查找这样的消息
### 使用.dockerignore文件
使用 Dockerfile 构建镜像时最好是将 Dockerfile 放置在一个新建的空目录下。然后将构建镜像所需要的文件添加到该目录中。为了提高构建镜像的效率，你可以在目录下新建一个.dockerignore文件来指定要忽略的文件和目录。.dockerignore 文件的排除模式语法和 Git 的 .gitignore 文件相似。
```docker
*.md
!README*.md
README-secret.md
```
### 使用多阶段构建
使用多阶段构建在 Docker 17.05 以上版本中，你可以使用 多阶段构建 来减少所构建镜像的大小.

如下只是一个简单地例子，并不能适合用在生产环境。

```Dockerfile
FROM 10.1.6.15/services/maven:3.6-jdk-8-alpine AS build
WORKDIR /code

RUN mkdir -p /root/.m2 && mkdir -p /code/.cache
COPY ./settings.xml /root/.m2/settings.xml

COPY ./ /code/
# 打包
RUN mvn --batch-mode package 
FROM  10.1.6.15/services/openjdk:8-jre-alpine
ENV TIME_ZONE=Asia/Shanghai \
    APP_HOME=/home/investment-plan \
    JVM_XMS="-Xms256m" \
    JVM_XMX="-Xmx2048m" \
    UID=1326 \
    GID=1326 \
    USER="investment-plan"
RUN mkdir -p $APP_HOME && addgroup  -g $GID $USER && adduser  -G $USER -u $UID   $USER --disabled-password 
RUN ln -snf /usr/share/zoneinfo/$TIME_ZONE /etc/localtime \
    && echo $TIME_ZONE > /etc/timezone
COPY --from=build /code/docker-startup.sh $APP_HOME/docker-startup.sh
COPY --from=build /code/services.jar $APP_HOME/services.jar

RUN chmod +x $APP_HOME/docker-startup.sh && chmod 750 -R /home && chown $USER:$USER -R /home
USER $USER
WORKDIR $APP_HOME
EXPOSE 9080
ENTRYPOINT ["sh","/home/investment-plan/docker-startup.sh"]
```

### 最小化镜像层数

在 Docker 17.05 甚至更早 1.10之 前，尽量减少镜像层数是非常重要的，不过现在的版本已经有了一定的改善了：

在 1.10 以后，只有 RUN、COPY 和 ADD 指令会创建层，其他指令会创建临时的中间镜像，但是不会直接增加构建的镜像大小了。
上节课我们也讲解到了 17.05 版本以后增加了多阶段构建的支持，允许我们把需要的数据直接复制到最终的镜像中，这就允许我们在中间阶段包含一些工具或者调试信息了，而且不会增加最终的镜像大小。
当然减少RUN、COPY、ADD的指令仍然是很有必要的，但是我们也需要在 Dockerfile 可读性（也包括长期的可维护性）和减少层数之间做一个平衡。

### LABEL
你可以给镜像添加标签来帮助组织镜像、记录许可信息、辅助自动化构建等。每个标签一行，由 LABEL 开头加上一个或多个标签对。
```dockerfile
# Set one or more individual labels
LABEL com.example.version="0.0.1-beta"
LABEL vendor="ACME Incorporated"
LABEL com.example.release-date="2023-02-08"
LABEL com.example.version.is-production="Canary"
```
一个镜像可以包含多个标签，在 1.10 之前，建议将所有标签合并为一条LABEL指令，以防止创建额外的层，但是现在这个不再是必须的了，以上内容也可以写成下面这样:
```dockerfile
# Set multiple labels at once, using line-continuation characters to break long lines
LABEL vendor=ACME\ Incorporated \
      com.example.is-production="" \
      com.example.version="Canary" \
      com.example.release-date="2023-02-09"
```
这些都是官方仓库给的一些示范: [例子](https://github.com/docker-library/docs)