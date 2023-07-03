# bibouroku-podman > operate-pod-by-kube-yaml

文書作成日:2023年03月12日 最終更新日:2023年07月02日

Podの生成などをyamlファイルで行う方法について記述する。これによって複数のコンテナを単一のコマンドで実行でき、必要なオプションなどを起動の度に指定することも不要になる。Dockerでは`docker compose`に相当するようだ。

1. [使用するイメージ](#使用するイメージ)
2. [使用するPod](#使用するpod)
3. [各コンテナの起動](#各コンテナの起動)
4. [Podの情報からyamlファイルの生成](#podの情報からyamlファイルの生成)
5. [yamlファイルからPodを生成](#yamlファイルからpodを生成)
6. [動作環境](#動作環境)
7. [参考文献](#参考文献)

## 使用するイメージ
イメージ名とタグ、入手に用いるコマンドを列挙する。

「PHPイメージの取得コマンド」
```
sudo podman pull php:fpm-alpine3.17
```

「Nginxイメージの取得コマンド」
```
sudo podman pull nginx:1.23.4-alpine3.17
```

「取得したイメージの確認」
```
$ sudo podman images
REPOSITORY                  TAG                   IMAGE ID      CREATED      SIZE
docker.io/library/php       fpm-alpine3.17        c31e8c80ffc5  7 days ago   77 MB
docker.io/library/nginx     1.23.4-alpine3.17     8e75cbc5b25c  7 days ago   42.8 MB
```

## 使用するPod
名称を`yaml-test`、Nginxが80番ポート使用するのでPod側の80番ポートを端末側の9001番ポートに割り当てるとする。

「Podの作成」
```
sudo podman pod create --name yaml-test -p 9001:80
```

「Podの詳細を確認」
```
$ sudo podman pod inspect yaml-test
{
     "Id": "0262e761c5b499b1ad78250d810594064f61cc61a524d72d58690e52e727ec2a",
     "Name": "yaml-test",
     "Created": "2023-04-07T00:16:09.383182584+09:00",
     "CreateCommand": [
          "podman",
          "pod",
          "create",
          "--name",
          "yaml-test",
          "-p",
          "9001:80"
     ],
     "State": "Created",
     "Hostname": "yaml-test",
     "CreateCgroup": true,
     "CgroupParent": "machine.slice",
     "CgroupPath": "machine.slice/machine-libpod_pod_0262e761c5b499b1ad78250d810594064f61cc61a524d72d58690e52e727ec2a.slice",
     "CreateInfra": true,
     "InfraContainerID": "8161d2dbc72fee76e83776673c112f5a97e4adb9666b86c206458fa05d1c3cc4",
     "InfraConfig": {
          "PortBindings": {
               "80/tcp": [
     {
          "HostIp": "",
          "HostPort": "9001"
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
               "Id": "8161d2dbc72fee76e83776673c112f5a97e4adb9666b86c206458fa05d1c3cc4",
               "Name": "0262e761c5b4-infra",
               "State": "configured"
          }
     ]
}
```

## 各コンテナの起動
NginxとPHPのコンテナをそれぞれ次のコマンドで起動する。

「Nginxコンテナの起動コマンド」
```
sudo podman run -dt --pod yaml-test --name nginx nginx:1.23.4-alpine3.17
```

「PHPコンテナの起動コマンド」
```
sudo podman run -dt --pod yaml-test --name php-fpm php:f
pm-alpine3.17
```

「Podの詳細を確認」
```
$ sudo podman pod inspect yaml-test
{
     "Id": "0262e761c5b499b1ad78250d810594064f61cc61a524d72d58690e52e727ec2a",
     "Name": "yaml-test",
     "Created": "2023-04-07T00:16:09.383182584+09:00",
     "CreateCommand": [
          "podman",
          "pod",
          "create",
          "--name",
          "yaml-test",
          "-p",
          "9001:80"
     ],
     "State": "Running",
     "Hostname": "yaml-test",
     "CreateCgroup": true,
     "CgroupParent": "machine.slice",
     "CgroupPath": "machine.slice/machine-libpod_pod_0262e761c5b499b1ad78250d810594064f61cc61a524d72d58690e52e727ec2a.slice",
     "CreateInfra": true,
     "InfraContainerID": "8161d2dbc72fee76e83776673c112f5a97e4adb9666b86c206458fa05d1c3cc4",
     "InfraConfig": {
          "PortBindings": {
               "80/tcp": [
     {
          "HostIp": "",
          "HostPort": "9001"
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
               "Id": "8161d2dbc72fee76e83776673c112f5a97e4adb9666b86c206458fa05d1c3cc4",
               "Name": "0262e761c5b4-infra",
               "State": "running"
          },
          {
               "Id": "b6d5dcfd6b6584a5edfb584b68684520c9a7afab3b3de33a9f8056fe089e81f6",
               "Name": "nginx",
               "State": "running"
          },
          {
               "Id": "e5e7824a13fc5693434a7136b98b06fee713fed0e33e2f45a2f182b2057bad66",
               "Name": "php-fpm",
               "State": "running"
          }
     ]
}

```

「Nginxコンテナの動作確認」
```
$ curl localhost:9001
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

## Podの情報からyamlファイルの生成
コンテナの情報含め、Podの情報をyamlファイルの出力するには以下のコマンドを使用する。

「Podからyamlファイルを生成するコマンド」
```
sudo podman generate kube yaml-test >> yaml-test.yaml
```

`>>`の左側に出力したいPodの名称を、右側に出力されるファイルの名称を記述する。この例の場合、コマンドの実行によって現在のディレクトリに`yaml-test.yaml`というファイルが生成される。

「yamlファイルの内容を確認」
```
$ ls
yaml-test.yaml

$ cat yaml-test.yaml
# Generation of Kubernetes YAML is still under development!
#
# Save the output of this file and use kubectl create -f to import
# it into Kubernetes.
#
# Created with podman-3.0.1
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2023-04-07T13:39:35Z"
  labels:
    app: yaml-test
  name: yaml-test
spec:
  containers:
  - args:
    - nginx
    - -g
    - daemon off;
    command:
    - /docker-entrypoint.sh
    env:
    - name: PATH
      value: /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
    - name: TERM
      value: xterm
    - name: container
      value: podman
    - name: NGINX_VERSION
      value: 1.23.4
    - name: PKG_RELEASE
      value: "1"
    - name: NJS_VERSION
      value: 0.7.11
    image: docker.io/library/nginx:1.23.4-alpine3.17
    name: nginx
    ports:
    - containerPort: 80
      hostPort: 9001
      protocol: TCP
    resources: {}
    securityContext:
      allowPrivilegeEscalation: true
      capabilities:
        drop:
        - CAP_MKNOD
        - CAP_NET_RAW
        - CAP_AUDIT_WRITE
      privileged: false
      readOnlyRootFilesystem: false
      seLinuxOptions: {}
    tty: true
    workingDir: /
  - args:
    - php-fpm
    command:
    - docker-php-entrypoint
    env:
    - name: PATH
      value: /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
    - name: TERM
      value: xterm
    - name: container
      value: podman
    - name: PHP_CFLAGS
      value: -fstack-protector-strong -fpic -fpie -O2 -D_LARGEFILE_SOURCE -D_FILE_OFFSET_BITS=64
    - name: PHP_CPPFLAGS
      value: -fstack-protector-strong -fpic -fpie -O2 -D_LARGEFILE_SOURCE -D_FILE_OFFSET_BITS=64
    - name: PHP_LDFLAGS
      value: -Wl,-O1 -pie
    - name: GPG_KEYS
      value: 39B641343D8C104B2B146DC3F9C39DC0B9698544 E60913E4DF209907D8E30D96659A97C9CF2A795A
        1198C0117593497A5EC5C199286AF1F9897469DC
    - name: PHP_URL
      value: https://www.php.net/distributions/php-8.2.4.tar.xz
    - name: PHP_ASC_URL
      value: https://www.php.net/distributions/php-8.2.4.tar.xz.asc
    - name: PHP_INI_DIR
      value: /usr/local/etc/php
    - name: PHP_VERSION
      value: 8.2.4
    - name: PHP_SHA256
      value: bc7bf4ca7ed0dd17647e3ea870b6f062fcb56b243bfdef3f59ff7f94e96176a8
    - name: PHPIZE_DEPS
      value: "autoconf \t\tdpkg-dev dpkg \t\tfile \t\tg++ \t\tgcc \t\tlibc-dev \t\tmake
        \t\tpkgconf \t\tre2c"
    image: docker.io/library/php:fpm-alpine3.17
    name: php-fpm
    resources: {}
    securityContext:
      allowPrivilegeEscalation: true
      capabilities:
        drop:
        - CAP_MKNOD
        - CAP_NET_RAW
        - CAP_AUDIT_WRITE
      privileged: false
      readOnlyRootFilesystem: false
      seLinuxOptions: {}
    tty: true
    workingDir: /var/www/html
  dnsConfig: {}
  restartPolicy: Never
status: {}
```

## yamlファイルからPodを生成
Podが生成されることを明らかにするために、ここでは一度すべてのPodを削除しておく。

「既存のPodを停止、削除してPod一覧の確認」
```
$ sudo podman pod stop -a
0262e761c5b499b1ad78250d810594064f61cc61a524d72d58690e52e727ec2a
$ sudo podman pod rm -a
0262e761c5b499b1ad78250d810594064f61cc61a524d72d58690e52e727ec2a
$ sudo podman pod ps
POD ID  NAME    STATUS  CREATED  INFRA ID  # OF CONTAINERS
```

用意したyamlファイルからPodを生成する時は、次のコマンドを実行する。`yaml-test.yaml`がPodの情報を含むファイルの名称である。

「yamlファイルからPodを生成するコマンド」
```
sudo podman play kube yaml-test.yaml
```

「Podの詳細を確認」
```
$ sudo podman pod ps
POD ID        NAME       STATUS   CREATED             INFRA ID      # OF CONTAINERS
e3998306fc09  yaml-test  Running  About a minute ago  22fba5a2eb66  3

$ sudo podman pod inspect yaml-test
{
     "Id": "e3998306fc095ef44fb89139e35168b1f06c60d7054d21df7a372ebad62b2665",
     "Name": "yaml-test",
     "Created": "2023-04-07T22:54:03.522330533+09:00",
     "State": "Running",
     "Hostname": "yaml-test",
     "Labels": {
          "app": "yaml-test"
     },
     "CreateCgroup": true,
     "CgroupParent": "machine.slice",
     "CgroupPath": "machine.slice/machine-libpod_pod_e3998306fc095ef44fb89139e35168b1f06c60d7054d21df7a372ebad62b2665.slice",
     "CreateInfra": true,
     "InfraContainerID": "22fba5a2eb66e59b939f3c0102f6135ceed279af7e75a733f4f607d7d465199c",
     "InfraConfig": {
          "PortBindings": {
               "80/tcp": [
     {
          "HostIp": "",
          "HostPort": "9001"
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
     "NumContainers": 3,
     "Containers": [
          {
               "Id": "22fba5a2eb66e59b939f3c0102f6135ceed279af7e75a733f4f607d7d465199c",
               "Name": "e3998306fc09-infra",
               "State": "running"
          },
          {
               "Id": "b445e91242f11bf5f6f3b008d53e0bdb0cc0248c776f367179c9cc510b831f7a",
               "Name": "yaml-test-php-fpm",
               "State": "running"
          },
          {
               "Id": "d63bc15c568a0d9ea6624eceb79712866a688030f22c44c125a522ad427e86a1",
               "Name": "yaml-test-nginx",
               "State": "running"
          }
     ]
}
```

Podの名称がコンテナ名の先頭に付加されるようだ。`--name`オプションの値に依存した情報を使用するときは注意が必要だろう。

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
  memFree: 2147708928
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
  uptime: 664h 37m 2.54s (Approximately 27.67 days)
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

## 参考文献
1. https://access.redhat.com/documentation/ja-jp/red_hat_enterprise_linux/8/html-single/building_running_and_managing_containers/index#proc_generating-a-yaml-file-using-podman_assembly_porting-containers-to-openshift-using-podman (コンテナーの構築、実行、および管理 Red Hat Enterprise Linux 8 | Red Hat Customer Portal)
2. https://access.redhat.com/documentation/ja-jp/red_hat_enterprise_linux/8/html-single/building_running_and_managing_containers/index#proc_automatically-running-containers-and-pods-using-podman_assembly_porting-containers-to-openshift-using-podman (コンテナーの構築、実行、および管理 Red Hat Enterprise Linux 8 | Red Hat Customer Portal)
