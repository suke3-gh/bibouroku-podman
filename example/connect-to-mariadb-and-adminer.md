# bibouroku-podman > connect-to-mariadb-and-adminer

文書作成日:2023年04月02日 最終更新日:2023年07月02日

ここでは関係データベース管理システムの`mariaDB`と、データベース内のコンテンツ管理ソフトウェアの`Adminer`をPodmanで使用する手順を書き留めておく。

1. [使用するイメージの取得](#使用するイメージの取得)
2. [使用するPodの作成](#使用するpodの作成)
3. [mariaDBコンテナの起動](#mariadbコンテナの起動)
4. [Adminerコンテナの起動](#adminerコンテナの起動)
5. [Podにコンテナが含まれているかを確認](#podにコンテナが含まれているかを確認)
6. [ブラウザからAdminerに接続](#ブラウザからadminerに接続)
7. [AdminerからmariaDBにログイン](#adminerからmariadbにログイン)
8. [動作環境](#動作環境)
9. [参考文献](#参考文献)

## 使用するイメージの取得
イメージはそれぞれ`docker.io`から取得する。コマンドは以下の通り。

「Adminerイメージの取得コマンド」
```
sudo podman pull adminer:4.8.1-standalone
```

「MariaDBのイメージ取得コマンド」
```
sudo podman pull mariadb:10.7.8
```

「取得したイメージの確認」
```
$ sudo podman images
REPOSITORY                 TAG                   IMAGE ID      CREATED      SIZE
docker.io/library/adminer  4.8.1-standalone      a5e1b3241b20  9 days ago   258 MB
docker.io/library/mariadb  10.7.8                0509cc86926d  2 weeks ago  402 MB
```

## 使用するPodの作成
Podの名称は`network-test`、使用するポートは`9001`とし、コマンドを実行。

「Podの作成コマンド」
```
sudo podman pod create --name network-test -p 9001:8080
```

「Podの確認」
```
$ sudo podman pod ps
POD ID        NAME          STATUS   CREATED        INFRA ID      # OF CONTAINERS
b8d869f0f795  network-test  Created  2 seconds ago  7bb90b864aba  1
```

## mariaDBコンテナの起動
コンテナの起動については`podman run`コマンドを用いるが、このときに使用するPodの指定が必要。またmariaDBがrootとしてログインする時のパスワードも必須なので、ここで設定しておく。Podを使用するためポート番号の指定は不要になる。使用するイメージはIDによって指定している。

「mariaDBコンテナの起動コマンド」
```
sudo podman run -d --pod network-test --name mariadb --env MARIADB_ROOT_PASSWORD=pass4mariadb 0509cc86926d
```

オプションなどの説明:
- `-d`: デタッチモードでの起動。
- `--pod`: コンテナが含まれることになるPodの名称。
- `--name`: コンテナの名称。データベースの場合ではこれがサーバ名になる。
- `--env`: 何らかの環境変数。ここでrootのパスワードを設定している。

コンテナが無事に起動しているか、またmariaDBはインストールされていて使用可能かを`podman exec`コマンドによって確認する。

「コンテナとmariaDBの動作確認」
```
$ sudo podman exec -it mariadb bash
root@network-test:/# maria db -u root -p
bash: maria: command not found
root@network-test:/# mariadb -u root -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 3
Server version: 10.7.8-MariaDB-1:10.7.8+maria~ubu2004 mariadb.org binary distribution

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> exit
Bye
root@network-test:/# exit
exit
```

## Adminerコンテナの起動
Adminerに関してもコマンドは、上記のmariaDBの場合とほぼ同様。差異はパスワードの設定が不要なので`--env`オプションがないこと。動作確認もmariaDBと同様に行った。

「Adminerコンテナの起動コマンド」
```
sudo podman run -d --pod network-test --name adminer a5e1b3241b20
```

「コンテナとAdminerの動作確認」
```
$ sudo podman exec -it adminer bash
adminer@network-test:/var/www/html$ ls
adminer.php  designs  index.php  plugin-loader.php  plugins  plugins-enabled
adminer@network-test:/var/www/html$ exit
exit
```

## Podにコンテナが含まれているかを確認
`podman pod inspect`コマンドでPodの内容を確認しておく。

「Podの詳細を確認」
```
$ sudo podman pod inspect network-test
{
     "Id": "b8d869f0f795f10309b2f1f52bbc1f6b9be591e7c5680709e19a327d33e45659",
     "Name": "network-test",
     "Created": "2023-04-02T00:05:52.203748428+09:00",
     "CreateCommand": [
          "podman",
          "pod",
          "create",
          "--name",
          "network-test",
          "-p",
          "9001:8080"
     ],
     "State": "Running",
     "Hostname": "network-test",

     -省略-

     "Containers": [
          {
               "Id": "4387ab4e0ccd65952f50394deaf6b61b2bb82cab4a2f7e82878c6fec32873f09",
               "Name": "adminer",
               "State": "running"
          },
          {
               "Id": "7bb90b864aba30723844220e9b1c4299b16a7ddc4ba3d58772505963c633cf3a",
               "Name": "b8d869f0f795-infra",
               "State": "running"
          },
          {
               "Id": "fc411ec8021a43235cee09dd00cdb80d38ff107c34c78414c37b7107a827459c",
               "Name": "mariadb",
               "State": "running"
          }
     ]
}
```

## ブラウザからAdminerに接続
Podに割り当てたポートに接続することでAdminerを使用できる。ここでは9001番ポートを割り当てしたので、次のURLを開けばよい。

「WebブラウザからAdminerを使用するためのURL」
```
localhost:9001
```

## AdminerからmariaDBにログイン
Adminerが表示されたら、各項目に情報を入力する。
- `データベース種類`: mariaDBを使用する場合、ここは`MySQL`でよい。
- `サーバ`: mariaDBのコンテナの名称とした`mariadb`を入力。
- `ユーザ名`: root用のパスワードしか設定していないので`root`とする。
- `パスワード`: mariaDBコンテナ起動の際に設定した`pass4mariadb`
- `データベース`: この項目は空欄でよい。

この段階でmariaDBにAdminer経由で接続出来るようになる。

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

## 参考文献
1. https://hub.docker.com/_/mariadb (mariadb - Official Image | Docker Hub)
2. https://hub.docker.com/_/adminer (adminer - Official Image | Docker Hub)
