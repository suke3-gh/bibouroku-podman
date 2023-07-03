# bibouroku-podman > setup-of-rootless-mode

文書作成日:2023年04月15日 最終更新日:2023年07月03日

公式の文書とほぼ同じ内容になるが、Podmanをrootlessモード、つまり`sudo`が不要でコマンドを実行できる状態にする手順を記す。

1. [必要なパッケージのインストール](#必要なパッケージのインストール)
2. [UIDの登録](#uidの登録)
3. [rootlessモード有効化の確認](#rootlessモード有効化の確認)
4. [rootless時のイメージの扱い](#rootless時のイメージの扱い)
5. [rootless時のコンテナの扱い](#rootless時のコンテナの扱い)
6. [動作環境](#動作環境)
7. [参考文献](#参考文献)

## 必要なパッケージのインストール
要求されるパッケージは`slirp4netns`と`fuse-overlayfs`の二つであり、それぞれ`apt install`コマンドでインストール可能。

「slirp4netnsのインストールコマンド」
```
sudo apt install slirp4netns
```

「fuse-overlayfsのインストールコマンド」
```
sudo apt install fuse-overlayfs
```

## UIDの登録
ここでのUIDとはユーザ識別子のことで、rootlessモードのPodmanは実行するユーザのUIDのリストを要求するようだ。UIDのリストは`/etc/subuid `と`/etc/subgid`というファイルに記載する。どちらファイルも以下のような内容を記述をする。

「subuidとsubgidファイルの記述内容」
```
username:100000:65536
test:165536:65536
```

`username`の部分には非root権限でPodmanを実行したいユーザ名を記述する。なお、対象ファイルをVimで開くコマンドは次の通り。

「subuidをvimで編集するコマンド」
```
sudo vim /etc/subuid
```

「subgidをvimで編集するコマンド」
```
sudo vim /etc/subgid
```

## rootlessモード有効化の確認
この段階でrootlessモードが有効化されているはずであり、その確認は`podman info`コマンドで行える。

「rootless動作の確認」
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
    - container_id: 1
      host_id: 100000
      size: 65536
    uidmap:
    - container_id: 0
      host_id: 1000
      size: 1
    - container_id: 1
      host_id: 100000
      size: 65536
  kernel: 5.10.159-20950-g3963226d9eb4
  linkmode: dynamic
  memFree: 2284601344
  memTotal: 2926096384
  ociRuntime:
    name: runc
    package: 'containerd.io: /usr/bin/runc'
    path: /usr/bin/runc
    version: |-
      runc version 1.1.5
      commit: v1.1.5-0-gf19387a
      spec: 1.0.2-dev
      go: go1.19.7
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
  -省略- 
```

security > rootlessの項がtrueになっているのでrootlessモードで動作しているとわかる。`podman info`コマンドを`sudo`を付加して実行してしまうと、この項がfalseと表示されてしまうことに注意。

## rootless時のイメージの扱い
rootlessモード取得したイメージは`sudo`、つまりrootからは表示されない。またその逆も同様のようだ。これは片方で取得したイメージは、もう片方で実行出来ないということになる。

「rootlessモードでのイメージ確認」
```
$ podman images
REPOSITORY                     TAG                   IMAGE ID      CREATED        SIZE
docker.io/library/python       3.9.16-slim-bullseye  dafea68fa71e  2 days ago     130 MB
docker.io/library/hello-world  latest                feb5d9fea6a5  18 months ago  19.9 kB
```

「rootからのイメージ確認」
```
$ sudo podman images
REPOSITORY                  TAG                 IMAGE ID      CREATED      SIZE
k8s.gcr.io/pause            3.2                 80d28bedfe5d  3 years ago  688 kB
```

## rootless時のコンテナの扱い
イメージと同様に、コンテナでも実行したユーザが異なるとそのコンテナは表示されなかった。

「rootlessモードでのコンテナ確認」
```
$ podman run -dt --name python python:3.9.16-slim-bullseye
786e3d6b59b87e8bf83a5e9f9469d035e052968ece037ba219cff00417d38bbb

$ podman ps
CONTAINER ID  IMAGE                                          COMMAND  CREATED        STATUS            PORTS   NAMES
786e3d6b59b8  docker.io/library/python:3.9.16-slim-bullseye  python3  7 seconds ago  Up 7 seconds ago          python
```

「rootからのコンテナ確認」
```
$ sudo podman ps
CONTAINER ID  IMAGE   COMMAND  CREATED  STATUS  PORTS   NAMES
```

実行ユーザが異なる場合は、コンテナ名称の重複が許されるようだ。次の例では`python`という名称でそれぞれ実行出来ている。

```
# 「コンテナ名称の重複確認」
$ podman ps
CONTAINER ID  IMAGE                                          COMMAND  CREATED        STATUS            PORTS   NAMES
786e3d6b59b8  docker.io/library/python:3.9.16-slim-bullseye  python3  4 minutes ago  Up 4 minutes ago          python

$ sudo podman ps
CONTAINER ID  IMAGE                                       COMMAND               CREATED        STATUS            PORTS   NAMES
6fde5a8182bd  docker.io/library/adminer:4.8.1-standalone  php -S [::]:8080 ...  5 seconds ago  Up 5 seconds ago          python
```

コンテナの削除において`-a`オプションを付加した場合、異なるユーザのコンテナに影響はなかった。


「-aオプションの適用範囲を確認」
```
$ sudo podman container rm -a -f

$ sudo podman ps
CONTAINER ID  IMAGE   COMMAND  CREATED  STATUS  PORTS   NAMES

$ podman ps
CONTAINER ID  IMAGE                                          COMMAND  CREATED         STATUS             PORTS   NAMES
786e3d6b59b8  docker.io/library/python:3.9.16-slim-bullseye  python3  43 minutes ago  Up 43 minutes ago          python
```

## 動作環境
「Linuxのバージョン情報」
```
$ cat /proc/version
Linux version 5.10.159-20950-g3963226d9eb4 (chrome-bot@chromeos-release-builder-us-central1-b-x32-6-h80v) (Chromium OS 15.0_pre465103_p20220825-r13 clang version 15.0.0 (/var/tmp/portage/sys-devel/llvm-15.0_pre465103_p20220825-r13/work/llvm-15.0_pre465103_p20220825/clang db1978b67431ca3462ad8935bf662c15750b8252), LLD 15.0.0) #1 SMP PREEMPT Mon Feb 20 18:10:10 PST 2023
```

「OSのバージョン情報」
```
$ cat /etc/os-release
PRETTY_NAME="Debian GNU/Linux 11 (bullseye)"
NAME="Debian GNU/Linux"
VERSION_ID="11"
VERSION="11 (bullseye)"
VERSION_CODENAME=bullseye
ID=debian
HOME_URL="https://www.debian.org/"
SUPPORT_URL="https://www.debian.org/support"
BUG_REPORT_URL="https://bugs.debian.org/"
```

「Podmanのバージョン情報」
```
$ podman -v
podman version 3.0.1
```

## 参考文献
1. https://github.com/containers/podman/blob/main/docs/tutorials/rootless_tutorial.md (podman/docs/tutorials/rootless_tutorial.md at main · containers/podman · GitHub)
2. https://opensource.com/article/19/2/how-does-rootless-podman-work (How does rootless Podman work? | Opensource.com)
