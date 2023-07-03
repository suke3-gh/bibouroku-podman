# bibouroku-podman > executable-file-not-found-in-path-oci-not-found

文書作成日:2023年03月22日 最終更新日:2023年6月30日

コンテナをPodに含める際に使う`podman run`コマンドで次のエラーが生じたので、解決までの過程や考察を記録する。

「発生したエラーメッセージ」
```
$ sudo podman run hello --pod pod-test
Error: runc create failed: unable to start container process: exec: "--pod": executable file not found in $PATH: OCI not found
```

1. [エラー文の考察](#エラー文の考察)
2. [検証:使用イメージの変更](#検証使用イメージの変更)
3. [検証:公式の導入手順を使用](#検証公式の導入手順を使用)
4. [検証:オプションの記述位置を変更](#検証オプションの記述位置を変更)
5. [結論](#結論)
6. [動作環境](#動作環境)

## エラー文の考察
パスの中に実行ファイルが存在しないとのことであり、`--pod`の部分に問題があると指摘されている。しかし他のオプションについて試すと、同様のエラーが生じた。

「引数の明示しての実行」
```
$ sudo podman run hello -d --pod "pod-test"
Error: runc create failed: unable to start container process: exec: "-d": executable file not found in $PATH: OCI not found
```

「--podの記述せずに実行」
```
$ sudo podman run hello -d
Error: runc create failed: unable to start container process: exec: "-d": executable file not found in $PATH: OCI not found
```

## 検証:使用イメージの変更
エラーメッセージで検索を行うと「bashがコンテナにインストールされていない」などの情報が多かったので、他のイメージを試す。しかし結果はエラーメッセージも含めて変化はなく、解決しなかった。

「イメージを変更しての実行」
```
$ sudo podman run debian:bullseye-slim -d
Error: runc create failed: unable to start container process: exec: "-d": executable file not found in $PATH: OCI not found
```

## 検証:公式の導入手順を使用
ランタイムなどの問題か切り分けるために、動作が保証されている公式文書上のコマンドを利用することにした。

公式の文書: https://docs.podman.io/en/latest/Introduction.html

busyboxのイメージを使用するコマンドを用いた結果、エラーが発生しなかった。

「公式の文書通りの実行」
```
$ sudo podman run -it docker.io/library/busybox
Trying to pull docker.io/library/busybox:latest...
Getting image source signatures
Copying blob 4b35f584bb4f done  
Copying config 7cfbbec896 done  
Writing manifest to image destination
Storing signatures
/ #
```

少なくともイメージに起因する問題ではないと言えるので、次に使用するオプションに着目する。

`-d`を用いるとエラーが生じた。

「-dオプションを付加して実行」
```
sudo podman run docker.io/library/busybox -d
Error: runc create failed: unable to start container process: exec: "-d": executable file not found in $PATH: OCI not found
```

`-p`でもエラーが発生する。

「-pオプションを付加して実行」
```
$ sudo podman run docker.io/library/busybox -p 8088:80
Error: runc create failed: unable to start container process: exec: "-p": executable file not found in $PATH: OCI not found
```

## 検証:オプションの記述位置を変更
公式文書のコマンドとの差異と対策2での検証から、コマンドにおけるオプションの記述位置の変更を試す。公式のコマンドのように`run`の直後でオプションを記述するとエラーメッセージが変わった。

「オプション記述位置を変更して実行」
```
$ sudo podman run -p:8088:80 docker.io/library/busybox
Error: must provide a non-empty container host IP to publish
```

`-d`ではエラーが発生しない。

「-dオプションを付加して実行」
```
$ sudo podman run -d docker.io/library/busybox
95bd8239993b21d1a9f2e27414cb9107e6cb369cb2e34f31936321b0126517c5
```

## 結論
上記三つの検証より、オプションの記述位置によってエラーが生じたと判断する。

`--pod`の位置を`run`の直後にすることで無事に動作した。

「オプションの記述位置を修正して実行」
```
$ sudo podman run --pod pod-test hello
!... Hello Podman World ...!

         .--"--.           
       / -     - \         
      / (O)   (O) \        
   ~~~| -=(,Y,)=- |         
    .---. /`  \   |~~      
 ~/  o  o \~~~~.----. ~~   
  | =(X)= |~  / (O (O) \   
   ~~~~~~~  ~| =(Y_)=-  |   
  ~~~~    ~~~|   U      |~~ 

Project:   https://github.com/containers/podman
Website:   https://podman.io
Documents: https://docs.podman.io
Twitter:   @Podman_io
```

「Podに含まれたかの確認」
```
$ sudo podman container ps
CONTAINER ID  IMAGE                 COMMAND  CREATED       STATUS                 PORTS   NAMES
6d6d61f09fc1  k8s.gcr.io/pause:3.2           45 hours ago  Up About a minute ago          71b289d2028f-infra

$ sudo podman pod inspect pod-test
{
     "Id": "71b289d2028f1c5a5557efaee6f352615311e199997f0ec2fa5b579876eb9e07",
     "Name": "pod-test",
     "Created": "2023-03-21T02:16:35.050191385+09:00",
     "CreateCommand": [
          "podman",
          "pod",
          "create",
          "--name",
          "pod-test"
     ],
     "State": "Degraded",
     "Hostname": "pod-test",
     "CreateCgroup": true,
     "CgroupParent": "machine.slice",
     "CgroupPath": "machine.slice/machine-libpod_pod_71b289d2028f1c5a5557efaee6f352615311e199997f0ec2fa5b579876eb9e07.slice",
     "CreateInfra": true,
     "InfraContainerID": "6d6d61f09fc1c86833314aa7ed0065c087b707c3cf5687d5b5e9c7699d7dcc72",
     "InfraConfig": {
          "PortBindings": {
               
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
     "NumContainers": 2,
     "Containers": [
          {
               "Id": "45a733b79d1c0f5e1524862a67cbd782a27c88fb89639683a0962b9bc45ab5d6",
               "Name": "epic_greider",
               "State": "exited"
          },
          {
               "Id": "6d6d61f09fc1c86833314aa7ed0065c087b707c3cf5687d5b5e9c7699d7dcc72",
               "Name": "71b289d2028f-infra",
               "State": "running"
          }
     ]
}
```

## 動作環境
|項目|内容|
|----|----|
|ハードウェア|ASUS C425TA|
|OS|Google ChromeOS 110.0.5481.181（Official Build） (64 ビット)|
|Podman|version 3.0.1|

「OSのバージョン情報」
```
$ cat /proc/version
Linux version 5.10.159-20950-g3963226d9eb4 (chrome-bot@chromeos-release-builder-us-central1-b-x32-6-h80v) (Chromium OS 15.0_pre465103_p20220825-r13 clang version 15.0.0 (/var/tmp/portage/sys-devel/llvm-15.0_pre465103_p20220825-r13/work/llvm-15.0_pre465103_p20220825/clang db1978b67431ca3462ad8935bf662c15750b8252), LLD 15.0.0) #1 SMP PREEMPT Mon Feb 20 18:10:10 PST 2023
```
