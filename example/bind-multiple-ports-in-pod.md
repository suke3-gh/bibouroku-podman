# bibouroku-podman > bind-multiple-ports-in-pod

文書作成日:2023年04月03日 最終更新日:2023年07月02日

ある一つのPodについて、複数のポート番号を割り当てる手順を書き留める。同一のPodに存在しつつ、個別にポート番号を割り当てるコンテナは`Nignx`と`Adminer`とする。

1. [使用するイメージの取得](#使用するイメージの取得)
2. [使用するPodの作成](#使用するpodの作成)
3. [各コンテナの起動](#各コンテナの起動)
4. [指定のポートに接続](#指定のポートに接続)
5. [動作環境](#動作環境)

## 使用するイメージの取得
各イメージを取得するコマンドは以下の通り。

「Nginxのイメージ取得コマンド」
```
sudo podman pull nginx:1.23.3
```

「Adminerのイメージ取得コマンド」
```
sudo podman pull adminer:4.8.1-standalone
```

「取得したイメージの確認」
```
$ sudo podman images
REPOSITORY                 TAG                   IMAGE ID      CREATED      SIZE
docker.io/library/nginx    1.23.3                ac232364af84  10 days ago  146 MB
docker.io/library/adminer  4.8.1-standalone      a5e1b3241b20  10 days ago  258 MB
k8s.gcr.io/pause           3.2                   80d28bedfe5d  3 years ago  688 kB
```

## 使用するPodの作成
Podを作成する際に、`-p`オプションを複数使用することで多数のポート番号を割り当てられる。コンテナのNginxは`80`番ポートを、Adminerは`8080`番ポートを使用するので、これらを端末側の`9001`と`9002`番ポートに割り当てるとする。Podの名称を「NetworkTest」とするとき、次のコマンドでPodを作成する。

「Podの作成コマンド」
```
sudo podman pod create --name NetworkTest -p 9001:80 -p 9002:8080
```

「Podの確認」
```
$ sudo podman pod inspect NetworkTest
{
     "Id": "e8c974b0aa93fccbb622e3513e058ac6930677e68128ff9910343d1f3e9b8b1c",
     "Name": "NetworkTest",
     "Created": "2023-04-03T00:42:05.462814309+09:00",
     "CreateCommand": [
          "podman",
          "pod",
          "create",
          "--name",
          "NetworkTest",
          "-p",
          "9001:80",
          "-p",
          "9002:8080"
     ],
     "State": "Created",
     "Hostname": "NetworkTest",
     "CreateCgroup": true,
     "CgroupParent": "machine.slice",
     "CgroupPath": "machine.slice/machine-libpod_pod_e8c974b0aa93fccbb622e3513e058ac6930677e68128ff9910343d1f3e9b8b1c.slice",
     "CreateInfra": true,
     "InfraContainerID": "c11cab8bb54365c9ecb995b2063279939984143b4368cc6a711bd617cfeb69ba",
     "InfraConfig": {
          "PortBindings": {
               "80/tcp": [
     {
          "HostIp": "",
          "HostPort": "9001"
     }
],
               "8080/tcp": [
     {
          "HostIp": "",
          "HostPort": "9002"
     }
]
          },
          "HostNetwork": false,
          "StaticIP": "",
          "StaticMAC": "",
          "NoManageResolvConf": false,
          "DNSServer": null,
          "DNSSearch": null,
          "DNSOption": null,
          "NoManageHosts": false,
          "HostAdd": null,
          "Networks": null,
          "NetworkOptions": null
     },
     "SharedNamespaces": [
          "uts",
          "ipc",
          "net"
     ],
     "NumContainers": 1,
     "Containers": [
          {
               "Id": "c11cab8bb54365c9ecb995b2063279939984143b4368cc6a711bd617cfeb69ba",
               "Name": "e8c974b0aa93-infra",
               "State": "configured"
          }
     ]
}
```

## 各コンテナの起動
Podの内部で各コンテナにポート番号を割り当てたので、コンテナを起動する際に`-p`オプションは不要。利便性のために`--name`オプションによってコンテナに名付けを行う。ここではイメージの指定にIDを使用している。

「Nginxコンテナを名付けして起動」
```
sudo podman run -d -t --pod NetworkTest --name nginx ac2
32364af84
```

「Adminerコンテナを名付けして起動」
```
sudo podman run -d -t --pod NetworkTest --name adminer a5e1b3241b20
```

「Podと各コンテナの状況を確認」
```
$ sudo podman pod inspect NetworkTest
{
     "Id": "57d955ba420a0acda1d82932e690630da9e0be15c44222eb06be19481d9754f4",
     "Name": "NetworkTest",
     "Created": "2023-04-03T00:43:21.573340952+09:00",
     "CreateCommand": [
          "podman",
          "pod",
          "create",
          "--name",
          "NetworkTest",
          "-p",
          "9001:80",
          "-p",
          "9002:8080"
     ],
     "State": "Running",
     "Hostname": "NetworkTest",
     "CreateCgroup": true,
     "CgroupParent": "machine.slice",
     "CgroupPath": "machine.slice/machine-libpod_pod_57d955ba420a0acda1d82932e690630da9e0be15c44222eb06be19481d9754f4.slice",
     "CreateInfra": true,
     "InfraContainerID": "02c3dd801cda9472019133421310662b9459cfe5a42c399c5b5c4efadb23e426",
     "InfraConfig": {
          "PortBindings": {
               "80/tcp": [
     {
          "HostIp": "",
          "HostPort": "9001"
     }
],
               "8080/tcp": [
     {
          "HostIp": "",
          "HostPort": "9002"
     }
]
          },
          "HostNetwork": false,
          "StaticIP": "",
          "StaticMAC": "",
          "NoManageResolvConf": false,
          "DNSServer": null,
          "DNSSearch": null,
          "DNSOption": null,
          "NoManageHosts": false,
          "HostAdd": null,
          "Networks": null,
          "NetworkOptions": null
     },
     "SharedNamespaces": [
          "ipc",
          "net",
          "uts"
     ],
     "NumContainers": 3,
     "Containers": [
          {
               "Id": "02c3dd801cda9472019133421310662b9459cfe5a42c399c5b5c4efadb23e426",
               "Name": "57d955ba420a-infra",
               "State": "running"
          },
          {
               "Id": "8aa465bc06f7f96bf753a0eb006d976e18dce70ff11032358479375c86dca27f",
               "Name": "adminer",
               "State": "running"
          },
          {
               "Id": "ad0db5b1e7bb4318979e77867fee4a9c637b769f5d4c144a9e07dae5e2a2f307",
               "Name": "nginx",
               "State": "running"
          }
     ]
}
```

## 指定のポートに接続
この段階でポート番号を個別に指定すると、各コンテナへ接続が可能になっている。それぞれ`localhost:9001`、`localhost:9002`にcurlやWebブラウザでアクセスすると動作の確認が出来る。

「curlによるNginxコンテナの動作確認」
```
$ curl localhost:9001
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>

-省略-
```

「curlによるAdminerコンテナの動作確認」
```
$ curl localhost:9002
<!DOCTYPE html>
<html lang="en" dir="ltr">
<meta http-equiv="Content-Type" content="text/html; charset=utf-8">
<meta name="robots" content="noindex">
<title>Login - Adminer</title>

-省略-
```

## 動作環境
「podman infoコマンドの実行結果」
```
$ podman info
host:
  arch: amd64
  buildahVersion: 1.19.6
  cgroupManager: cgroupfs
  cgroupVersion: v1
  conmon:
    package: 'conmon: /usr/bin/conmon'
    path: /usr/bin/conmon
    version: 'conmon version 2.0.25, commit: unknown'
  cpus: 4
  distribution:
    distribution: debian
    version: "11"
  eventLogger: journald
  hostname: penguin
  idMappings:
    gidmap:
    - container_id: 0
      host_id: 1000
      size: 1
    uidmap:
    - container_id: 0
      host_id: 1000
      size: 1
  kernel: 5.10.159-20950-g3963226d9eb4
  linkmode: dynamic
  memFree: 2039119872
  memTotal: 2926096384
  ociRuntime:
    name: runc
    package: 'containerd.io: /usr/bin/runc'
    path: /usr/bin/runc
    version: |-
      runc version 1.1.4
      commit: v1.1.4-0-g5fd4c4d
      spec: 1.0.2-dev
      go: go1.19.6
      libseccomp: 2.5.1
  os: linux
  remoteSocket:
    exists: true
    path: /run/user/1000/podman/podman.sock
  security:
    apparmorEnabled: false
    capabilities: CAP_CHOWN,CAP_DAC_OVERRIDE,CAP_FOWNER,CAP_FSETID,CAP_KILL,CAP_NET_BIND_SERVICE,CAP_SETFCAP,CAP_SETGID,CAP_SETPCAP,CAP_SETUID,CAP_SYS_CHROOT
    rootless: true
    seccompEnabled: true
    selinuxEnabled: false
  slirp4netns:
    executable: /usr/bin/slirp4netns
    package: 'slirp4netns: /usr/bin/slirp4netns'
    version: |-
      slirp4netns version 1.0.1
      commit: 6a7b16babc95b6a3056b33fb45b74a6f62262dd4
      libslirp: 4.4.0
  swapFree: 0
  swapTotal: 0
  uptime: 549h 54m 25.27s (Approximately 22.88 days)
registries:
  docker.io:
    Blocked: false
    Insecure: false
    Location: docker.io
    MirrorByDigestOnly: false
    Mirrors: null
    Prefix: docker.io
  search:
  - quay.io
  - docker.io
store:
  configFile: /home/suke3/.config/containers/storage.conf
  containerStore:
    number: 0
    paused: 0
    running: 0
    stopped: 0
  graphDriverName: overlay
  graphOptions:
    overlay.mount_program:
      Executable: /usr/bin/fuse-overlayfs
      Package: 'fuse-overlayfs: /usr/bin/fuse-overlayfs'
      Version: |-
        fusermount3 version: 3.10.3
        fuse-overlayfs: version 1.4
        FUSE library version 3.10.3
        using FUSE kernel interface version 7.31
  graphRoot: /home/suke3/.local/share/containers/storage
  graphStatus:
    Backing Filesystem: btrfs
    Native Overlay Diff: "false"
    Supports d_type: "true"
    Using metacopy: "false"
  imageStore:
    number: 1
  runRoot: /run/user/1000/containers
  volumePath: /home/suke3/.local/share/containers/storage/volumes
version:
  APIVersion: 3.0.0
  Built: 0
  BuiltTime: Thu Jan  1 09:00:00 1970
  GitCommit: ""
  GoVersion: go1.15.15
  OsArch: linux/amd64
  Version: 3.0.1
```
