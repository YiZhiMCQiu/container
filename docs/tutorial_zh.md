# 教程

通过构建、运行和发布一个简单的 Web 服务器镜像，带你体验 `container` 的使用之旅。

## 试用 `container` CLI

启动应用，并尝试一些基础命令，熟悉命令行工具（CLI）。

### 启动 container 服务

启动 `container` 所需的服务：

```bash
container system start
```

如果你尚未安装 Linux 内核，命令会提示你安装：

<pre>
% container system start

Verifying apiserver is running...
Installing base container filesystem...
No default kernel configured.
Install the recommended default kernel from [https://github.com/kata-containers/kata-containers/releases/download/3.17.0/kata-static-3.17.0-arm64.tar.xz]? [Y/n]: y
Installing kernel...
%
</pre>

然后，通过列出所有容器来验证应用是否正常运行：

```bash
container list --all
```

如果你还没有创建任何容器，命令会输出一个空列表：

<pre>
% container list --all
ID  IMAGE  OS  ARCH  STATE  ADDR
%
</pre>

### 获取 CLI 帮助

你可以在任何 `container` CLI 命令后加上 `--help` 选项来获取帮助：

<pre>
% container --help
OVERVIEW: A container platform for macOS

USAGE: container [--debug] <subcommand>

OPTIONS:
  --debug                 Enable debug output [environment: CONTAINER_DEBUG]
  --version               Show the version.
  -h, --help              Show help information.

CONTAINER SUBCOMMANDS:
  create                  Create a new container
  delete, rm              Delete one or more containers
  exec                    Run a new command in a running container
  inspect                 Display information about one or more containers
  kill                    Kill one or more running containers
  list, ls                List containers
  logs                    Fetch container stdio or boot logs
  run                     Run a container
  start                   Start a container
  stop                    Stop one or more running containers

IMAGE SUBCOMMANDS:
  build                   Build an image from a Dockerfile
  images, image, i        Manage images
  registry, r             Manage registry configurations

SYSTEM SUBCOMMANDS:
  builder                 Manage an image builder instance
  system, s               Manage system components

%
</pre>

### 命令缩写

你可以通过缩写命令和选项来减少输入。例如，将 `container list` 缩写为 `container ls`，将 `--all` 缩写为 `-a`：

<pre>
% container ls -a
ID  IMAGE  OS  ARCH  STATE  ADDR
%
</pre>

使用 `--help` 标志可以查看有哪些缩写可用。

### 配置本地 DNS 域（可选）

`container` 内置了 DNS 服务，简化了对容器化应用的访问。如果你想为本教程配置名为 `test` 的本地域名，请运行：

```bash
sudo container system dns create test
container system dns default set test
```

根据提示输入管理员密码。第一个命令需要管理员权限，因为它会在 `/etc/resolver` 目录下创建域名配置文件，并让 macOS DNS 解析器重新加载配置。

第二个命令将 `test` 设为默认域名。例如，如果默认域是 `test`，你用 `--name my-web-server` 启动容器后，访问 `my-web-server.test` 就会返回该容器的 IP 地址。

## 构建镜像

为一个简单 Python Web 服务器设置 `Dockerfile`，并用它构建名为 `web-test` 的容器镜像。

### 创建简单项目

开启终端，创建名为 `web-test` 的目录，用于存放构建镜像所需的文件：

```bash
mkdir web-test
cd web-test
```

在 `web-test` 目录下创建名为 `Dockerfile` 的文件，内容如下：

```dockerfile
FROM docker.io/python:alpine
WORKDIR /content
RUN apk add curl
RUN echo '<!DOCTYPE html><html><head><title>Hello</title></head><body><h1>Hello, world!</h1></body></html>' > index.html
CMD ["python3", "-m", "http.server", "80", "--bind", "0.0.0.0"]
```

- `FROM` 指定以 Python 3 的 alpine 版本为基础镜像。
- `WORKDIR` 在镜像中创建 `/content` 目录，并设为当前目录。
- 第一个 `RUN` 安装 `curl` 命令，第二个 `RUN` 创建简单的 HTML 首页 `/content/index.html`。
- `CMD` 配置容器启动时，在 80 端口运行一个 Python Web 服务器。服务器监听 `0.0.0.0`，允许主机和其他容器访问。

### 构建 Web 服务器镜像

运行如下命令，用 `Dockerfile` 构建名为 `web-test` 的镜像：

```bash
container build --tag web-test --file Dockerfile .
```

最后的 `.` 表示构建上下文为当前目录（`web-test`）。你可以用 `COPY` 命令把上下文内容复制进镜像。

构建完成后，列出所有镜像。你会看到基础镜像和你刚构建的镜像：

<pre>
% container images list
NAME      TAG     DIGEST
python    alpine  b4d299311845147e7e47c970...
web-test  latest  25b99501f174803e21c58f9c...
%
</pre>

## 运行容器

使用你构建的镜像，运行 Web 服务器并体验各种交互方式。

### 启动 Web 服务器

用 `container run` 启动名为 `my-web-server` 的容器：

```bash
container run --name my-web-server --detach --rm web-test
```

- `--detach` 让容器在后台运行，方便继续在当前终端操作。
- `--rm` 在容器停止后自动删除。

此时列出容器，会看到 `my-web-server` 和构建镜像时自动启动的容器。注意 `ADDR` 列显示的 IP，如 `192.168.64.3`：

<pre>
% container ls
ID             IMAGE                                               OS     ARCH   STATE    ADDR
buildkit       ghcr.io/apple/container-builder-shim/builder:0.0.3  linux  arm64  running  192.168.64.2
my-web-server  web-test:latest                                     linux  arm64  running  192.168.64.3
%
</pre>

用该 IP 打开网站：

```bash
open http://192.168.64.3
```

如果前面配置了 `test` 本地域，可以用完整主机名访问：

```bash
open http://my-web-server.test
```

### 在容器内运行其他命令

你可以用 `container exec` 在 `my-web-server` 容器中运行其他命令。例如，用 `ls` 列出 `/content` 目录下的文件：

<pre>
% container exec my-web-server ls /content
index.html
%
</pre>

如果想在容器内进一步操作，可以启动 shell：

<pre>
% container exec --tty --interactive my-web-server sh
/content # ls
index.html
/content # uname -a
Linux my-web-server 6.12.28 #1 SMP Tue May 20 15:19:05 UTC 2025 aarch64 Linux
/content # exit
%
</pre>

- `--tty` 让容器内 shell 认为输入是终端设备。
- `--interactive` 把主机终端的输入连接到容器 shell。

这两个选项常常缩写为 `-ti` 或 `-it`。

### 从另一个容器访问 Web 服务器

你的 Web 服务器不仅主机可访问，其他容器也能访问。用你的 `web-test` 镜像再启动一个容器，运行 `curl` 获取第一个容器的 `index.html`：

```bash
container run -it --rm web-test curl http://192.168.64.3
```

输出应为：

<pre>
% container run -it --rm web-test curl http://192.168.64.3
&lt;!DOCTYPE html>&lt;html>&lt;head>&lt;title>Hello&lt;/title>&lt;/head>&lt;body>&lt;h1>Hello, world!&lt;/h1>&lt;/body>&lt;/html>
%
</pre>

如果设置了 `test` 域名，也可以这样访问：

```bash
container run -it --rm web-test curl http://my-web-server.test
```

## 使用已发布的镜像

将你的镜像推送到容器仓库，发布后你和他人都可以使用。

### 发布 Web 服务器镜像

要发布镜像，需要推送到注册表服务。通常需要先认证。假设你在 `registry.example.com` 有账号，用户名 `fido`，密码或 token 为 `my-secret`，个人仓库名同用户名。

> [!NOTE]
> 默认情况下 `container` 使用 Docker Hub。  
> 你可以通过 `container registry default set <registry url>` 更换默认仓库。  
> 详细命令可查 `container registry` 子命令。

登录注册表：

```bash
container registry login {registry.example.com}
```

为你的镜像添加一个带仓库和用户名的新标签：

```bash
container images tag web-test {registry.example.com/fido}/web-test:latest
```

然后推送镜像：

```bash
container images push {registry.example.com/fido}/web-test:latest
```

### 拉取并运行已发布镜像

检验你的已发布镜像，先停止当前容器并删除镜像，然后用远程镜像运行：

```bash
container stop my-web-server
container images delete web-test {registry.example.com/fido}/web-test:latest
container run --name my-web-server --detach --rm {registry.example.com/fido}/web-test:latest
```

## 清理

停止容器并关闭应用。

### 关闭 Web 服务器

停止你的 Web 服务器容器：

```bash
container stop my-web-server
```

此时如果列出所有容器，会看到你用 `--rm` 选项运行的容器已自动删除：

<pre>
% container list --all
ID        IMAGE                                               OS     ARCH   STATE    ADDR
buildkit  ghcr.io/apple/container-builder-shim/builder:0.0.3  linux  arm64  running  192.168.64.2
%
</pre>

### 停止 container 服务

要彻底关闭 `container`，运行：

```bash
container system stop
```
