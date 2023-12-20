# docker  构建其他cpu架构镜像

国产化服务器提供的多数是arm版本的linux环境

## docker 启动build 构建多cpu架构 镜像

```
/etc/docker/daemon.json
{
  "features": {
    "buildkit": true
  }
}
systemctl restart docker 重启docker
```

## docker 1.19及高版本 启用buildx

注意 从 Docker Engine 版本 23.0.0 开始，Buildx 在单独的包中分发：`docker-buildx-plugin`。 在早期版本中，Buildx 包含在 `docker-ce-cli` 包中。 当您升级到此版本的 Docker Engine 时，请确保更新所有 包。例如，在 Ubuntu 上：

```console
$ sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose
```

由于 docker 1.23 开始 buildx 从 docker工具独立出来，所以1.24之后的版本 我们要单独安装  https://github.com/docker/buildx

1.19版本是由测试buildx 但是依然可以通过一下方式来启用buildx

```
mkdir -p ~/.docker/cli-plugins 
mv buildx-v0.12.0.linux-amd64 ~/.docker/cli-plugins/docker-buildx
chmod +x ~/.docker/cli-plugins/docker-buildx
systemctl restrat docker
docker buildx install #将 docker builder 命令设置为 docker buildx build 的别名。这 导致能够docker build使用当前的buildx构建器。要删除此别名，请运行 docker buildx uninstall。

[root@control-plane ~]# docker buildx version  #看到版本则安装成功
github.com/docker/buildx v0.12.0 542e5d810e4a1a155684f5f3c5bd7e797632a12f
```

构建方式：

```
docker buildx create --use --name mybuilder #创建名为 mybuilder 的构建器
docker buildx ls  #查看构建器
NAME/NODE    DRIVER/ENDPOINT             STATUS   BUILDKIT PLATFORMS
container    docker-container                              
  container0 unix:///var/run/docker.sock inactive          
mybuilder *  docker-container                              
  mybuilder0 unix:///var/run/docker.sock running  v0.12.1  linux/amd64, linux/amd64/v2, linux/amd64/v3, linux/386
default      docker                                        
  default    default                     running  v0.11.6  linux/amd64, linux/amd64/v2, linux/amd64/v3, linux/386

docker buildx build --builder mybuilder --platform linux/arm64/v3 -t test:armv3 --output type=docker . #构建 
```

