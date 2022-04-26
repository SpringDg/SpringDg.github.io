---
title: Gitlab配合Gitlab-runner实现CI/CD
date: 2022-04-20 07:17:13
tags: CICD
---

## 安装和配置Gitlab-runner
使用如下命令启动gitlab-runner容器
```
docker run -d --name gitlab-runner --restart always \
     -v /srv/gitlab-runner/config:/etc/gitlab-runner \
     -v /var/run/docker.sock:/var/run/docker.sock \
     gitlab/gitlab-runner:latest
```
注册runner，直接用命令把参数配置全部补全
```
docker exec -it gitlab-runner gitlab-runner register -n \
   --url https://gitlab.com/ \
   --registration-token QJiAZYz3KSJyhWfsHKhC \
   --executor docker \
   --tag-list "test-runner" \
   --description "test-runner" \
   --docker-image "docker:stable" \
   --docker-volumes /var/run/docker.sock:/var/run/docker.sock
```
或者根据提示把参数配置补全
```
docker exec -it gitlab-runner gitlab-runner register
```

## 创建gitlab-ci.yml文件
在git的根目录创建gitlab-ci.yml
```
testjob:
  script:
    - docker ps
    - sh ./check.sh # check的shell文件是用来删除已经存在（上次编译）的容器和镜像
    - cd LotteryAnalysis.BackEnd
    - docker build -t lotteryserver .
    - docker run -d --name lotteryserver -p 5000:5000 lotteryserver
  tags:
    - test-runner # 需要和注册的runner的tag一致才能触发runner运行，可以配合多个runner的tag进行CI/CD
```
根据自己的项目路径创建Dockerfile
```
# Dockerfile
FROM mcr.microsoft.com/dotnet/sdk AS build-env
WORKDIR /code
COPY *.csproj ./
RUN dotnet restore
COPY . ./
RUN dotnet publish -c Release -o out

FROM mcr.microsoft.com/dotnet/aspnet
WORKDIR /app
COPY --from=build-env /code/out ./

EXPOSE 5000

ENTRYPOINT ["dotnet", "LotteryAnalysisServer.dll"]
```
check.sh的内容
```
# check.sh
if [ $(docker ps -a --format {{.Names}} | grep lotteryserver) ]
then
    docker rm -f lotteryserver
    docker rmi lotteryserver
fi
```