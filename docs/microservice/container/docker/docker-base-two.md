# Docker的基本操作
## 镜像
### 镜像拉取
镜像的拉取主要是使用docker pull命令，从Docker镜像仓库获取镜像。很多相关的优质镜像可以访问 https://hub.docker.com/ 或者采用docker search命令

对应的格式为：
```shell
docker pull [OPTIONS] NAME[:TAG|@DIGEST]

Options:
  -a, --all-tags                Download all tagged images in the repository
      --disable-content-trust   Skip image verification (default true)
      --platform string         Set platform if server is multi-platform capable
  -q, --quiet                   Suppress verbose output
```

我们以作官方镜像hello-world作为示例
```shell
docker pull library/hello-world:latest
```
这样下载的就是hello-world latest标签的镜像。这里的library下的都是官方镜像用户名，对于不给出用户名的，默认为library。标签latest也可以不给出，默认为latest。所以上面的命令等价于：
```shell
docker pull hello-world
```
执行结果中也可以看得出来
```shell
Using default tag: latest
latest: Pulling from library/hello-world
2db29710123e: Pull complete 
Digest: sha256:2498fce14358aa50ead0cc6c19990fc6ff866ce72aeb5546e1d59caac3d0d60f
Status: Downloaded newer image for hello-world:latest
```
如果我们想要拉取别的标签版本可以进行直接指定
```shell
docker pull hello-world:nanoserver
```

### 镜像相关操作
镜像拉取完毕之后，相关多数操作可使用docker image完成。
```shell
Usage:  docker image COMMAND

Manage images

Commands:
  build       Build an image from a Dockerfile
  history     Show the history of an image
  import      Import the contents from a tarball to create a filesystem image
  inspect     Display detailed information on one or more images
  load        Load an image from a tar archive or STDIN
  ls          List images
  prune       Remove unused images
  pull        Pull an image or a repository from a registry
  push        Push an image or a repository to a registry
  rm          Remove one or more images
  save        Save one or more images to a tar archive (streamed to STDOUT by default)
  tag         Create a tag TARGET_IMAGE that refers to SOURCE_IMAGE
```

### 列出镜像
命令格式
```shell
Usage:  docker image ls [OPTIONS] [REPOSITORY[:TAG]]

List images

Aliases:
  ls, list

Options:
  -a, --all             Show all images (default hides intermediate images)
      --digests         Show digests
  -f, --filter filter   Filter output based on conditions provided
      --format string   Pretty-print images using a Go template
      --no-trunc        Don't truncate output
  -q, --quiet           Only show image IDs
```
如果正常想查看所有非中间层镜像的所有镜像直接使用 docker image ls 命令即可，如果想要看到中间层镜像可以加上 -a 参数，中间层镜像是docker为了加速镜像构建、重复利用资源的产物，是其它镜像所依赖的镜像。这些无标签镜像不应该删除，否则会导致上层镜像因为依赖丢失而出错。实际上，这些镜像也没必要删除，因为之前说过，相同的层只会存一遍，而这些镜像是别的镜像的依赖，因此并不会因为它们被列出来而多存了一份，无论如何你也会需要它们。只要删除那些依赖它们的镜像后，这些依赖的中间层镜像也会被连带删除。
```shell
docker image ls -a
```
使用普通 docker image ls 结果
```shell
docker image ls

REPOSITORY                          TAG       IMAGE ID       CREATED         SIZE
nginx                               latest    605c77e624dd   10 months ago   141MB
mysql                               latest    3218b38490ce   10 months ago   516MB
apacherocketmq/rocketmq-dashboard   latest    eae6c5db5d11   12 months ago   738MB
hello-world                         latest    feb5d9fea6a5   13 months ago   13.3kB
rocketmqinc/rocketmq                latest    09bbc30a03b6   3 years ago     380MB
```
列表包含了仓库名、标签、镜像 ID、创建时间以及所占用的空间。

使用 docker image ls hello-world 只展示hello-world镜像相关信息
```shell
docker image ls hello-world
REPOSITORY    TAG       IMAGE ID       CREATED         SIZE
hello-world   latest    feb5d9fea6a5   13 months ago   13.3kB
```

在此之上还可以结合 --f 参数进行一些过滤筛选。如 docker image ls -f since=hello-world:latest 可以看到比hello-world:latest更晚创建的所有镜像，而before则意味着之前。
```shell
docker image ls -f since=hello-world:latest 
REPOSITORY                          TAG       IMAGE ID       CREATED         SIZE
nginx                               latest    605c77e624dd   10 months ago   141MB
mysql                               latest    3218b38490ce   10 months ago   516MB
apacherocketmq/rocketmq-dashboard   latest    eae6c5db5d11   12 months ago   738MB
```

### 删除镜像

命令格式
```shell
Usage:  docker image rm [OPTIONS] IMAGE [IMAGE...]

Remove one or more images

Aliases:
  rm, rmi, remove

Options:
  -f, --force      Force removal of the image
      --no-prune   Do not delete untagged parents
```

对于我们的hello-world
```shell
REPOSITORY    TAG       DIGEST                                                                    IMAGE ID       CREATED         SIZE
hello-world   latest    sha256:2498fce14358aa50ead0cc6c19990fc6ff866ce72aeb5546e1d59caac3d0d60f   feb5d9fea6a5   13 months ago   13.3kB
```
我们可以采用以下命令都可以进行删除：
```shell
docker image rm hello-world:latest

docker image rm feb5d9fea6a5
```


## 容器
### 容器相关操作
容器相关操作主要使用 docker container 命令进行
```shell
Usage:  docker container COMMAND

Manage containers

Commands:
  attach      Attach local standard input, output, and error streams to a running container
  commit      Create a new image from a container's changes
  cp          Copy files/folders between a container and the local filesystem
  create      Create a new container
  diff        Inspect changes to files or directories on a container's filesystem
  exec        Run a command in a running container
  export      Export a container's filesystem as a tar archive
  inspect     Display detailed information on one or more containers
  kill        Kill one or more running containers
  logs        Fetch the logs of a container
  ls          List containers
  pause       Pause all processes within one or more containers
  port        List port mappings or a specific mapping for the container
  prune       Remove all stopped containers
  rename      Rename a container
  restart     Restart one or more containers
  rm          Remove one or more containers
  run         Run a command in a new container
  start       Start one or more stopped containers
  stats       Display a live stream of container(s) resource usage statistics
  stop        Stop one or more running containers
  top         Display the running processes of a container
  unpause     Unpause all processes within one or more containers
  update      Update configuration of one or more containers
  wait        Block until one or more containers stop, then print their exit codes
```

### 容器启动
启动容器有两种方式，一种是基于镜像新建一个容器并启动，另外一个是将在终止状态（stopped）的容器重新启动。

基于镜像新建一个容器并启动使用 docker container run 命令，也可以不写container，以hello-world为例，--name 参数是为容器指定名称
```shell
docker container run --name hworld hello-world
```

命令 docker container run 相关格式
```shell
Usage:  docker container run [OPTIONS] IMAGE [COMMAND] [ARG...]

Run a command in a new container

Options:
--add-host list                  Add a custom host-to-IP mapping (host:ip)
-a, --attach list                    Attach to STDIN, STDOUT or STDERR
--blkio-weight uint16            Block IO (relative weight), between 10 and 1000, or 0 to disable (default 0)
--blkio-weight-device list       Block IO weight (relative device weight) (default [])
--cap-add list                   Add Linux capabilities
--cap-drop list                  Drop Linux capabilities
--cgroup-parent string           Optional parent cgroup for the container
--cgroupns string                Cgroup namespace to use (host|private)
'host':    Run the container in the Docker host's cgroup namespace
'private': Run the container in its own private cgroup namespace
'':        Use the cgroup namespace as configured by the
default-cgroupns-mode option on the daemon (default)
--cidfile string                 Write the container ID to the file
--cpu-period int                 Limit CPU CFS (Completely Fair Scheduler) period
--cpu-quota int                  Limit CPU CFS (Completely Fair Scheduler) quota
--cpu-rt-period int              Limit CPU real-time period in microseconds
--cpu-rt-runtime int             Limit CPU real-time runtime in microseconds
-c, --cpu-shares int                 CPU shares (relative weight)
--cpus decimal                   Number of CPUs
--cpuset-cpus string             CPUs in which to allow execution (0-3, 0,1)
--cpuset-mems string             MEMs in which to allow execution (0-3, 0,1)
-d, --detach                         Run container in background and print container ID
--detach-keys string             Override the key sequence for detaching a container
--device list                    Add a host device to the container
--device-cgroup-rule list        Add a rule to the cgroup allowed devices list
--device-read-bps list           Limit read rate (bytes per second) from a device (default [])
--device-read-iops list          Limit read rate (IO per second) from a device (default [])
--device-write-bps list          Limit write rate (bytes per second) to a device (default [])
--device-write-iops list         Limit write rate (IO per second) to a device (default [])
--disable-content-trust          Skip image verification (default true)
--dns list                       Set custom DNS servers
--dns-option list                Set DNS options
--dns-search list                Set custom DNS search domains
--domainname string              Container NIS domain name
--entrypoint string              Overwrite the default ENTRYPOINT of the image
-e, --env list                       Set environment variables
--env-file list                  Read in a file of environment variables
--expose list                    Expose a port or a range of ports
--gpus gpu-request               GPU devices to add to the container ('all' to pass all GPUs)
--group-add list                 Add additional groups to join
--health-cmd string              Command to run to check health
--health-interval duration       Time between running the check (ms|s|m|h) (default 0s)
--health-retries int             Consecutive failures needed to report unhealthy
--health-start-period duration   Start period for the container to initialize before starting health-retries countdown (ms|s|m|h) (default 0s)
--health-timeout duration        Maximum time to allow one check to run (ms|s|m|h) (default 0s)
--help                           Print usage
-h, --hostname string                Container host name
--init                           Run an init inside the container that forwards signals and reaps processes
-i, --interactive                    Keep STDIN open even if not attached
--ip string                      IPv4 address (e.g., 172.30.100.104)
--ip6 string                     IPv6 address (e.g., 2001:db8::33)
--ipc string                     IPC mode to use
--isolation string               Container isolation technology
--kernel-memory bytes            Kernel memory limit
-l, --label list                     Set meta data on a container
--label-file list                Read in a line delimited file of labels
--link list                      Add link to another container
--link-local-ip list             Container IPv4/IPv6 link-local addresses
--log-driver string              Logging driver for the container
--log-opt list                   Log driver options
--mac-address string             Container MAC address (e.g., 92:d0:c6:0a:29:33)
-m, --memory bytes                   Memory limit
--memory-reservation bytes       Memory soft limit
--memory-swap bytes              Swap limit equal to memory plus swap: '-1' to enable unlimited swap
--memory-swappiness int          Tune container memory swappiness (0 to 100) (default -1)
--mount mount                    Attach a filesystem mount to the container
--name string                    Assign a name to the container
--network network                Connect a container to a network
--network-alias list             Add network-scoped alias for the container
--no-healthcheck                 Disable any container-specified HEALTHCHECK
--oom-kill-disable               Disable OOM Killer
--oom-score-adj int              Tune host's OOM preferences (-1000 to 1000)
--pid string                     PID namespace to use
--pids-limit int                 Tune container pids limit (set -1 for unlimited)
--platform string                Set platform if server is multi-platform capable
--privileged                     Give extended privileges to this container
-p, --publish list                   Publish a container's port(s) to the host
-P, --publish-all                    Publish all exposed ports to random ports
--pull string                    Pull image before running ("always"|"missing"|"never") (default "missing")
--read-only                      Mount the container's root filesystem as read only
--restart string                 Restart policy to apply when a container exits (default "no")
--rm                             Automatically remove the container when it exits
--runtime string                 Runtime to use for this container
--security-opt list              Security Options
--shm-size bytes                 Size of /dev/shm
--sig-proxy                      Proxy received signals to the process (default true)
--stop-signal string             Signal to stop a container (default "SIGTERM")
--stop-timeout int               Timeout (in seconds) to stop a container
--storage-opt list               Storage driver options for the container
--sysctl map                     Sysctl options (default map[])
--tmpfs list                     Mount a tmpfs directory
-t, --tty                            Allocate a pseudo-TTY
--ulimit ulimit                  Ulimit options (default [])
-u, --user string                    Username or UID (format: <name|uid>[:<group|gid>])
--userns string                  User namespace to use
--uts string                     UTS namespace to use
-v, --volume list                    Bind mount a volume
--volume-driver string           Optional volume driver for the container
--volumes-from list              Mount volumes from the specified container(s)
-w, --workdir string                 Working directory inside the container
```
这其中常常使用到的有以下参数
```text
-d 指定容器运行于前台还是后台，默认为false
-p 指定容器暴露的端口
-v 给容器挂载存储卷，挂载到容器的某个目录
-e 指定环境变量，容器中可以使用该环境变量
-m 指定容器的内存上限
-c 设置容器CPU权重，在CPU共享场景使用
--name 指定容器名字，后续可以通过名字进行容器管理
--net 容器网络设置
--restart 指定容器停止后的重启策略
```

容器重新启动使用 docker container start 命令
```shell
docker container start hworld
```

### 容器终止
容器终止使用命令 docker container stop 命令，相应的格式为
```shell
Usage:  docker container stop [OPTIONS] CONTAINER [CONTAINER...]

Stop one or more running containers

Options:
  -t, --time int   Seconds to wait for stop before killing it (default 10)
```
以刚才启动的hello-world为例，按照容器名称进行停止
```shell
docker container stop hworld
```


### 容器进入
某些时候需要进入容器进行操作，可以使用 docker attach 命令或 docker exec 命令。
```shell
docker container attach hworld

docker container -it exec hworld bash
```
命令 docker exec 使用参数 -it 时可以看到我们熟悉的 Linux 命令提示符。同时在打开的stdin中exit，对于attach而言会导致容器的停止，而对于exec则不会，所以更推荐使用exec。

### 容器删除
容器删除使用命令 docker container rm 命令，相应的格式为
```shell
Usage:  docker container rm [OPTIONS] CONTAINER [CONTAINER...]

Remove one or more containers

Options:
-f, --force     Force the removal of a running container (uses SIGKILL)
-l, --link      Remove the specified link
-v, --volumes   Remove anonymous volumes associated with the container
```

例如
```shell
docker container rm hworld
```
如果要删除一个运行中的容器，可以添加 -f 参数。Docker 会发送 SIGKILL 信号给容器。使用 docker container prune 可以删除所有处于终止状态的容器。
```shell
docker container prune
```


## 仓库
### 官方仓库
#### 登录或退出
使用命令 docker login 和 docker logout 可以进行DockerHub的登录和退出。

#### 搜索仓库
使用命令 docker search 可以按照关键字查找仓库镜像，如
```shell
docker search hello-world
NAME                                       DESCRIPTION                                     STARS     OFFICIAL   AUTOMATED
hello-world                                Hello World! (an example of minimal Dockeriz…   1898      [OK]       
kitematic/hello-world-nginx                A light-weight nginx container that demonstr…   153                  
tutum/hello-world                          Image to test docker deployments. Has Apache…   90                   [OK]
dockercloud/hello-world                    Hello World!                                    19                   [OK]
crccheck/hello-world                       Hello World web server in under 2.5 MB          15                   [OK]
vad1mo/hello-world-rest                    A simple REST Service that echoes back all t…   5                    [OK]
rancher/hello-world                                                                        4                    
ppc64le/hello-world                        Hello World! (an example of minimal Dockeriz…   2                    
thomaspoignant/hello-world-rest-json       This project is a REST hello-world API to bu…   2                    
ansibleplaybookbundle/hello-world-db-apb   An APB which deploys a sample Hello World! a…   2                    [OK]
ansibleplaybookbundle/hello-world-apb      An APB which deploys a sample Hello World! a…   1                    [OK]
strimzi/hello-world-producer                                                               0                    
armswdev/c-hello-world                     Simple hello-world C program on Alpine Linux…   0                    
strimzi/hello-world-consumer                                                               0                    
businessgeeks00/hello-world-nodejs                                                         0                    
koudaiii/hello-world                                                                       0                    
freddiedevops/hello-world-spring-boot                                                      0                    
strimzi/hello-world-streams                                                                0                    
garystafford/hello-world                   Simple hello-world Spring Boot service for t…   0                    [OK]
tacc/hello-world                                                                           0                    
tsepotesting123/hello-world                                                                0                    
kevindockercompany/hello-world                                                             0                    
dandando/hello-world-dotnet                                                                0                    
okteto/hello-world                                                                         0                    
rsperling/hello-world3                                                                     0                    
```

#### 拉取镜像
使用命令 docker pull 之前已经讲过。

#### 推送镜像
用户也可以在登录后通过 docker push 命令来将自己的镜像推送到 Docker Hub。 以hello-world为例，usrname需要改成自己的用户名称
```shell
 docker tag hello-world usrname/hello-world:latest
 
 docker push usrname/hello-world:latest
```
成功或使用 docker search命令即可搜索到自己的镜像内容
```shell
docker search usrname

NAME                         DESCRIPTION   STARS     OFFICIAL   AUTOMATED
usrname/rocketmq-dashboard                 0  
```