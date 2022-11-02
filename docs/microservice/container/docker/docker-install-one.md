## Docker的安装

- 获取快速安装脚本
  - ```text
     curl -fsSL get.docker.com -o get-docker.sh
    ```
- 快捷脚本安装docker
  - ```text
     sudo sh get-docker.sh --mirror Aliyun
    ```
- 出现错误
  - Invalid configuration value: failovermethod=priority in /etc/yum.repos.d/CentOS-epel.repo; Configuration: OptionBinding with id "failovermethod" does not exist
  - 这里需要去CentOS-epel.repo文件将配置项 failovermethod=priority 给注释掉
  - 使用VIM命令修改即可
    - ```text
       vim /etc/yum.repos.d/CentOS-epel.repo
      ```
- 再次使用快捷脚本安装
  - sudo sh get-docker.sh --mirror Aliyun
- 接着出现如下内容，没有报错相关的即可

```text
# Executing docker install script, commit: 93d2499759296ac1f9c510605fef85052a2c32be
Warning: the "docker" command appears to already exist on this system.

If you already have Docker installed, this script can cause trouble, which is
why we're displaying this warning and provide the opportunity to cancel the
installation.

If you installed the current Docker package using this script and are using it
again to update Docker, you can safely ignore this message.

You may press Ctrl+C now to abort this script.
+ sleep 20
+ sh -c 'yum install -y -q yum-utils'
+ sh -c 'yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo'
Adding repo from: https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
+ '[' stable '!=' stable ']'
+ sh -c 'yum makecache'
CentOS-8 - AppStream                                                                                                                                                                                     733 kB/s | 4.3 kB     00:00  
CentOS-8 - Base                                                                                                                                                                                          797 kB/s | 3.9 kB     00:00  
CentOS-8 - Extras                                                                                                                                                                                        283 kB/s | 1.5 kB     00:00  
Extra Packages for Enterprise Linux 8 - x86_64                                                                                                                                                           134 kB/s | 4.7 kB     00:00  
Docker CE Stable - x86_64                                                                                                                                                                                 18 kB/s | 3.5 kB     00:00  
Metadata cache created.
+ '[' -n '' ']'
+ sh -c 'yum install -y -q docker-ce'
+ version_gte 20.10
+ '[' -z '' ']'
+ return 0
+ sh -c 'yum install -y -q docker-ce-rootless-extras'

================================================================================

To run Docker as a non-privileged user, consider setting up the
Docker daemon in rootless mode for your user:

    dockerd-rootless-setuptool.sh install

Visit https://docs.docker.com/go/rootless/ to learn about rootless mode.


To run the Docker daemon as a fully privileged service, but granting non-root
users access, refer to https://docs.docker.com/go/daemon-access/

WARNING: Access to the remote API on a privileged Docker daemon is equivalent
         to root access on the host. Refer to the 'Docker daemon attack surface'
         documentation for details: https://docs.docker.com/go/attack-surface/

================================================================================
```

## 验证是否安装成功

- 使用help命令
  - docker --help
- 正常展示docker相关帮助内容即可，到此安装准备工作就算完成了

```text
Usage:  docker [OPTIONS] COMMAND

A self-sufficient runtime for containers

Options:
      --config string      Location of client config files (default "/root/.docker")
  -c, --context string     Name of the context to use to connect to the daemon (overrides DOCKER_HOST env var and default context set with "docker context use")
  -D, --debug              Enable debug mode
  -H, --host list          Daemon socket(s) to connect to
  -l, --log-level string   Set the logging level ("debug"|"info"|"warn"|"error"|"fatal") (default "info")
      --tls                Use TLS; implied by --tlsverify
      --tlscacert string   Trust certs signed only by this CA (default "/root/.docker/ca.pem")
      --tlscert string     Path to TLS certificate file (default "/root/.docker/cert.pem")
      --tlskey string      Path to TLS key file (default "/root/.docker/key.pem")
      --tlsverify          Use TLS and verify the remote
  -v, --version            Print version information and quit

Management Commands:
  app*        Docker App (Docker Inc., v0.9.1-beta3)
  builder     Manage builds
  buildx*     Docker Buildx (Docker Inc., v0.9.1-docker)
  config      Manage Docker configs
  container   Manage containers
  context     Manage contexts
  image       Manage images
  manifest    Manage Docker image manifests and manifest lists
  network     Manage networks
  node        Manage Swarm nodes
  plugin      Manage plugins
  scan*       Docker Scan (Docker Inc., v0.21.0)
  secret      Manage Docker secrets
  service     Manage services
  stack       Manage Docker stacks
  swarm       Manage Swarm
  system      Manage Docker
  trust       Manage trust on Docker images
  volume      Manage volumes

Commands:
  attach      Attach local standard input, output, and error streams to a running container
  build       Build an image from a Dockerfile
  commit      Create a new image from a container's changes
  cp          Copy files/folders between a container and the local filesystem
  create      Create a new container
  diff        Inspect changes to files or directories on a container's filesystem
  events      Get real time events from the server
  exec        Run a command in a running container
  export      Export a container's filesystem as a tar archive
  history     Show the history of an image
  images      List images
  import      Import the contents from a tarball to create a filesystem image
  info        Display system-wide information
  inspect     Return low-level information on Docker objects
  kill        Kill one or more running containers
  load        Load an image from a tar archive or STDIN
  login       Log in to a Docker registry
  logout      Log out from a Docker registry
  logs        Fetch the logs of a container
  pause       Pause all processes within one or more containers
  port        List port mappings or a specific mapping for the container
  ps          List containers
  pull        Pull an image or a repository from a registry
  push        Push an image or a repository to a registry
  rename      Rename a container
  restart     Restart one or more containers
  rm          Remove one or more containers
  rmi         Remove one or more images
  run         Run a command in a new container
  save        Save one or more images to a tar archive (streamed to STDOUT by default)
  search      Search the Docker Hub for images
  start       Start one or more stopped containers
  stats       Display a live stream of container(s) resource usage statistics
  stop        Stop one or more running containers
  tag         Create a tag TARGET_IMAGE that refers to SOURCE_IMAGE
  top         Display the running processes of a container
  unpause     Unpause all processes within one or more containers
  update      Update configuration of one or more containers
  version     Show the Docker version information
  wait        Block until one or more containers stop, then print their exit codes

Run 'docker COMMAND --help' for more information on a command.

To get more help with docker, check out our guides at https://docs.docker.com/go/guides/
```


## 启动docker
- 启动docker命令
  - ```text
       sudo systemctl enable docker
    ```
    - 这里会看到相关执行结果
      - Created symlink /etc/systemd/system/multi-user.target.wants/docker.service → /usr/lib/systemd/system/docker.service.
  - ```text
       sudo systemctl start docker
    ```


## 验证是否正常安装并启动成功
- 一般都是采用拉取hello-world镜像进行验证
- 使用命令run，如果没有拉取或hello-world镜像的话就会自动帮你拉取
  - ```text
       docker run hello-world
    ```
  - 看到以下内容就是正确安装并启动成功了
  
```text
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
2db29710123e: Pull complete 
Digest: sha256:2498fce14358aa50ead0cc6c19990fc6ff866ce72aeb5546e1d59caac3d0d60f
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```
