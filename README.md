# _containerd the hard way_
container runtime중 하나인 containerd를 어렵게😅 설치하는 방법입니다. 하이타워님의 [kubernetes the hard way](https://github.com/kelseyhightower/kubernetes-the-hard-way) 스타일로😄 

## 버전
2022년 2월 10일 기준 아래 최신 버전으로 설치 합니다.
- containerd: https://github.com/containerd/containerd/releases/tag/v1.5.9
- runc: https://github.com/opencontainers/runc/releases/tag/v1.1.0
- cni: https://github.com/containernetworking/plugins/releases/tag/v1.0.1
- crictl: https://github.com/kubernetes-sigs/cri-tools/releases/tag/v1.23.0
- nerdctl: https://github.com/containerd/nerdctl/releases/tag/v0.16.1

## 설치
### 다운로드
패키지 매니저를 사용하는 것이 아닌 타르볼을 다운로드 합니다.
```sh
wget -q --show-progress --https-only --timestamping \
  https://github.com/containerd/containerd/releases/download/v1.5.9/containerd-1.5.9-linux-amd64.tar.gz \
  https://github.com/opencontainers/runc/releases/download/v1.1.0/runc.amd64 \
  https://github.com/containernetworking/plugins/releases/download/v1.0.1/cni-plugins-linux-amd64-v1.0.1.tgz \
  https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.21.0/crictl-v1.21.0-linux-amd64.tar.gz \
  https://github.com/containerd/nerdctl/releases/download/v0.16.1/nerdctl-0.16.1-linux-amd64.tar.gz
```
### 디렉토리
설치시 필요한 디렉토리를 만들어 놓습니다.
```sh
sudo mkdir -p \
  /etc/containerd \
  /etc/cni/net.d \
  /opt/cni/bin \
  /etc/nerdctl
```
### 설치
압축을 풀어놓는 정도 입니다.
```sh
{
  sudo tar -xvf containerd-1.5.9-linux-amd64.tar.gz -C /usr/local/
  chmod +x runc.amd64
  sudo mv runc.amd64 /usr/local/bin/runc
  sudo tar -xvf cni-plugins-linux-amd64-v1.0.1.tgz -C /opt/cni/bin/
  sudo tar -xvf crictl-v1.21.0-linux-amd64.tar.gz -C /usr/local/bin/
  sudo tar -xvf nerdctl-0.16.1-linux-amd64.tar.gz -C /usr/local/bin/
  sudo rm -f /usr/local/bin/containerd-rootless*
}
```
### containerd 설정
containerd config 명령으로 기본 설정파일을 생성합니다. (옵션 조정이 필요한 경우 해당 파일을 수정하면 됩니다.)
```sh
containerd config default | sudo tee /etc/containerd/config.toml
```
### containerd.service 설정
containerd를 systemd에 등록합니다.
```sh
cat <<EOF | sudo tee /etc/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target
 
[Service]
ExecStartPre=/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd
Restart=always
RestartSec=5
Delegate=yes
KillMode=process
OOMScoreAdjust=-999
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity
 
[Install]
WantedBy=multi-user.target
EOF
```
### cni 설정
container network plugin 을 설정합니다. (기본 bridge 모드의 예시입니다. 아래에서 subnet은 임의로 지정된 값입니다.)
```sh
cat <<EOF | sudo tee /etc/cni/net.d/10-bridge.conf
{
    "cniVersion": "0.4.0",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cni0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "10.22.0.0/16"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
EOF
 
cat <<EOF | sudo tee /etc/cni/net.d/99-loopback.conf
{
    "cniVersion": "0.4.0",
    "name": "lo",
    "type": "loopback"
}
EOF
```
### crictl 설정
containerd 조작도구인 crictl 에 대한 설정입니다. containerd 를 사용하니 endpoint를 containerd로 설정합니다.
```sh
cat <<EOF | sudo tee /etc/crictl.yaml
runtime-endpoint: unix:///var/run/containerd/containerd.sock
image-endpoint: unix:///var/run/containerd/containerd.sock
timeout: 10
debug: false
EOF
```
### nerdctl 설정
nerdctl 에 대한 설정을 합니다. crictl이 단순 debug 도구라면 nerdctl 은 기존 docker 인터페이스에 가깝습니다. 
```sh

cat <<EOF | sudo tee /etc/nerdctl/nerdctl.toml
debug             = false
debug_full        = false
address           = "unix:///var/run/containerd/containerd.sock"
namespace         = "k8s.io"
snapshotter       = "native"
cni_path          = "/opt/cni/bin"
cni_netconfpath   = "/etc/cni/net.d"
cgroup_manager    = "cgroupfs"
insecure_registry = false
hosts_dir         = ["/etc/containerd/certs.d"]
EOF
```
👍
## containerd with tools
### crictl
```sh
#-----------------------------
# 버전
#-----------------------------
$ sudo crictl version
Version:  0.1.0
RuntimeName:  containerd
RuntimeVersion:  v1.5.9
RuntimeApiVersion:  v1alpha2
 
#-----------------------------
# 이미지 다운로드 (docker pull 처럼)
#-----------------------------
$ sudo crictl pull busybox:latest
 
#-----------------------------
# 이미지 조회 (docker images 처럼)
#-----------------------------
$ sudo crictl images
IMAGE                                   TAG                 IMAGE ID            SIZE
docker.io/library/busybox               latest              ec3f0931a6e6b       777kB
 
#-----------------------------
# 컨테이너 실행 (docker run 처럼)
# 기존 도커보다 조금 복잡합니다.
#-----------------------------
$ cat <<EOF | tee container-config.json
{
  "metadata": {
      "name": "busybox"
  },
  "image":{
      "image": "busybox"
  },
  "command": [
      "top"
  ],
  "log_path":"busybox.0.log",
  "linux": {
  }
}
EOF
$ cat <<EOF | tee pod-config.json
{
    "metadata": {
        "name": "nginx-sandbox",
        "namespace": "default",
        "attempt": 1,
        "uid": "hdishd83djaidwnduwk28bcsb"
    },
    "log_directory": "/tmp",
    "linux": {
    }
}
EOF
 
$ sudo crictl run container-config.json pod-config.json
 
#-----------------------------
# 컨테이너 리스트 조회(docker ps 처럼)
#-----------------------------
$ sudo crictl ps
CONTAINER           IMAGE               CREATED             STATE               NAME                ATTEMPT             POD ID
9efc813bac57d       busybox             2 hours ago         Running             busybox             0                   32fd267838c97
 
#-----------------------------
# pod 컨테이너 조회 (docker ps 처럼)
#-----------------------------
$ sudo crictl pods
POD ID              CREATED             STATE               NAME                NAMESPACE           ATTEMPT             RUNTIME
32fd267838c97       2 hours ago         Ready               nginx-sandbox       default             1                   (default)
 
 
#-----------------------------
# 컨테이너 접속(docker exec 처럼)
#-----------------------------
$ sudo crictl exec -it 9efc813bac57d /bin/sh
 
#-----------------------------
# 기타 가능한 command
#-----------------------------
$ sudo crictl -h
NAME:
   crictl - client for CRI
 
USAGE:
   crictl [global options] command [command options] [arguments...]
 
VERSION:
   v1.21.0
 
COMMANDS:
   attach              Attach to a running container
   create              Create a new container
   exec                Run a command in a running container
   version             Display runtime version information
   images, image, img  List images
   inspect             Display the status of one or more containers
   inspecti            Return the status of one or more images
   imagefsinfo         Return image filesystem info
   inspectp            Display the status of one or more pods
   logs                Fetch the logs of a container
   port-forward        Forward local port to a pod
   ps                  List containers
   pull                Pull an image from a registry
   run                 Run a new container inside a sandbox
   runp                Run a new pod
   rm                  Remove one or more containers
   rmi                 Remove one or more images
   rmp                 Remove one or more pods
   pods                List pods
   start               Start one or more created containers
   info                Display information of the container runtime
   stop                Stop one or more running containers
   stopp               Stop one or more running pods
   update              Update one or more running containers
   config              Get and set crictl client configuration options
   stats               List container(s) resource usage statistics
   completion          Output shell completion code
   help, h             Shows a list of commands or help for one command
```
### nerdctl
```sh
#-----------------------------
# 버전
#-----------------------------
$ sudo nerdctl version
Client:
 Version:       v0.16.1
 Git commit:    c4bd56b3aa220db037cc6c0a4e0c8cc062f2cc4c
 
Server:
 containerd:
  Version:      v1.5.9
  GitCommit:    1407cab509ff0d96baa4f0eb6ff9980270e6e620
 
#-----------------------------
# 이미지 다운로드 (docker pull 처럼)
#-----------------------------
$ sudo nerdctl pull nginx:latest
docker.io/library/nginx:latest:                                                   resolved       |++++++++++++++++++++++++++++++++++++++|
index-sha256:2834dc507516af02784808c5f48b7cbe38b8ed5d0f4837f16e78d00deb7e7767:    exists         |++++++++++++++++++++++++++++++++++++++|
manifest-sha256:bb129a712c2431ecce4af8dde831e980373b26368233ef0f3b2bae9e9ec515ee: exists         |++++++++++++++++++++++++++++++++++++++|
config-sha256:c316d5a335a5cf324b0dc83b3da82d7608724769f6454f6d9a621f3ec2534a5a:   exists         |++++++++++++++++++++++++++++++++++++++|
elapsed: 2.8 s                                                                    total:   0.0 B (0.0 B/s)
 
#-----------------------------
# 이미지 조회 (docker images 처럼)
#-----------------------------
$ sudo nerdctl images
REPOSITORY                               TAG                                                                 IMAGE ID        CREATED           PLATFORM       SIZE
nginx                                    latest                                                              2834dc507516    52 minutes ago    linux/amd64    823.4 MiB
 
 
#-----------------------------
# 컨테이너 실행 (docker run 처럼)
#-----------------------------
$ sudo nerdctl run -d -p 80:80 --name nginx-text nginx:latest
e547923573abba2bdd3afa559f98a22490b226897c2ef5322348f54f8f405b90
 
#-----------------------------
# 컨테이너 리스트 조회(docker ps 처럼)
#-----------------------------
$ sudo nerdctl ps
CONTAINER ID    IMAGE                               COMMAND                   CREATED           STATUS    PORTS                 NAMES
e547923573ab    docker.io/library/nginx:latest      "/docker-entrypoint.…"    33 seconds ago    Up        0.0.0.0:80->80/tcp    nginx-text
 
 
#-----------------------------
# 컨테이너 접속(docker exec 처럼)
#-----------------------------
$ sudo nerdctl exec -it nginx-text /bin/sh
 
#-----------------------------
# 기타 가능한 command
#-----------------------------
$ sudo nerdctl -h
nerdctl is a command line interface for containerd
 
Config file ($NERDCTL_TOML): /etc/nerdctl/nerdctl.toml
 
Usage:
  nerdctl [flags]
  nerdctl [command]
Management commands:
  apparmor    Manage AppArmor profiles
  container   Manage containers
  image       Manage images
  ipfs        Distributing images on IPFS
  namespace   Manage containerd namespaces
  network     Manage networks
  system      Manage containerd
  volume      Manage volumes
Commands:
  build       Build an image from a Dockerfile. Needs buildkitd to be running.
  commit      Create a new image from a container's changes
  completion  Generate the autocompletion script for the specified shell
  compose     Compose
  create      Create a new container. Optionally specify "ipfs://" or "ipns://" scheme to pull image from IPFS.
  events      Get real time events from the server
  exec        Run a command in a running container
  help        Help about any command
  images      List images
  info        Display system-wide information
  inspect     Return low-level information on objects.
  kill        Kill one or more running containers
  load        Load an image from a tar archive or STDIN
  login       Log in to a Docker registry
  logout      Log out from a Docker registry
  logs        Fetch the logs of a container. Currently, only containers created with `nerdctl run -d` are supported.
  pause       Pause all processes within one or more containers
  port        List port mappings or a specific mapping for the container
  ps          List containers
  pull        Pull an image from a registry. Optionally specify "ipfs://" or "ipns://" scheme to pull image from IPFS.
  push        Push an image or a repository to a registry. Optionally specify "ipfs://" or "ipns://" scheme to push image to IPFS.
  restart     Restart one or more running containers
  rm          Remove one or more containers
  rmi         Remove one or more images
  run         Run a command in a new container. Optionally specify "ipfs://" or "ipns://" scheme to pull image from IPFS.
  save        Save one or more images to a tar archive (streamed to STDOUT by default)
  start       Start one or more running containers
  stats       Display a live stream of container(s) resource usage statistics.
  stop        Stop one or more running containers
  tag         Create a tag TARGET_IMAGE that refers to SOURCE_IMAGE
  top         Display the running processes of a container
  unpause     Unpause all processes within one or more containers
  update      Update one or more running containers
  version     Show the nerdctl version information
  wait        Block until one or more containers stop, then print their exit codes.
```
