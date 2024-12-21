# Docker容器使用



| 命令 | 功能 | 示例 |
| ---- | ---- | ---- |
| `docker run` | 启动一个新的容器并运行命令 | `docker run -d ubuntu` |
| `docker ps` | 列出当前正在运行的容器 | `docker ps` |
| `docker ps -a` | 列出所有容器（包括已停止的容器） | `docker ps -a` |
| `docker build` | 使用Dockerfile构建镜像 | `docker build -t my-image.` |
| `docker images` | 列出本地存储的所有镜像 | `docker images` |
| `docker pull` | 从Docker仓库拉取镜像 | `docker pull ubuntu` |
| `docker push` | 将镜像推送到Docker仓库 | `docker push my-image` |
| `docker exec` | 在运行的容器中执行命令 | `docker exec -it container_name bash` |
| `docker stop` | 停止一个或多个容器 | `docker stop container_name` |
| `docker start` | 启动已停止的容器 | `docker start container_name` |
| `docker restart` | 重启一个容器 | `docker restart container_name` |
| `docker rm` | 删除一个或多个容器 | `docker rm container_name` |
| `docker rmi` | 删除一个或多个镜像 | `docker rmi my-image` |
| `docker logs` | 查看容器的日志 | `docker logs container_name` |
| `docker inspect` | 获取容器或镜像的详细信息 | `docker inspect container_name` |
| `docker exec -it` | 进入容器的交互式终端 | `docker exec -it container_name /bin/bash` |
| `docker network ls` | 列出所有Docker网络 | `docker network ls` |
| `docker volume ls` | 列出所有Docker卷 | `docker volume ls` |
| `docker-compose up` | 启动多容器应用（从`docker-compose.yml`文件） | `docker-compose up` |
| `docker-compose down` | 停止并删除由`docker-compose`启动的容器、网络等 | `docker-compose down` |
| `docker info` | 显示Docker系统的详细信息 | `docker info` |
| `docker version` | 显示Docker客户端和守护进程的版本信息 | `docker version` |
| `docker stats` | 显示容器的实时资源使用情况 | `docker stats` |
| `docker login` | 登录Docker仓库 | `docker login` |
| `docker logout` | 登出Docker仓库 | `docker logout` |

- -d：后台运行容器，例如 docker run -d ubuntu。
- -it：以交互式终端运行容器，例如 docker exec -it container_name bash。
- -t：为镜像指定标签，例如 docker build -t my-image .
- -P：是容器内部端口随机映射到主机的端口。
- -p：是容器内部端口绑定到指定的主机端口。
- --name：为容器指定名称。
- --rm：容器退出时自动清理容器内部的文件系统。
- -h HOSTNAME 或者 --hostname=HOSTNAME： 设定容器的主机名，它会被写到容器内的 /etc/hostname 和 /etc/hosts。
- --dns=IP_ADDRESS： 添加 DNS 服务器到容器的 /etc/resolv.conf 中，让容器用这个服务器来解析所有不在 /etc/hosts 中的主机名
- --net==net_mode： 指定容器的网络模式。

## 用dockerfile构建镜像

docker build -t iamge_name .
- -t 指定镜像名称
- .dockerfile所在目录

## docker-compose

docker-compose up 构建镜像并启动容器
- -d 后台运行

docker-compose down 停止并删除容器

docker-compose restart 重启容器

docker-compose start 启动容器

docker-compose stop 停止容器

## 保存镜像
docker save -o image_name.tar image_name

## 加载镜像
docker load -i image_name.tar

## 推送镜像
docker push image_name

## 拉取镜像
docker pull image_name
