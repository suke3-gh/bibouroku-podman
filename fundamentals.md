# bibouroku-podman > fundamentals

文書作成日:2023年03月12日 最終更新日:2023年6月30日

Linux上で動作するコンテナ管理ツールである`Podman`について、基本的な事を記す。OSはDebianを使用してると仮定する。

1. [インストール](#インストール)
2. [各コマンドについて](#各コマンドについて)
3. [Containerfileを用いたイメージの作成](#containerfileを用いたイメージの作成)
4. [Podについて](#podについて)
    1. [Podの作成](#podの作成)
    2. [作成したPodの詳細](#作成したpodの詳細)
    3. [Podの中でコンテナを起動](#podの中でコンテナを起動)
    4. [Podから指定のコンテナを取り除く](#podから指定のコンテナを取り除く)
    5. [Podに接続](#podに接続)
5. [参考文献](#参考文献)

## インストール
パッケージは公式で用意されているため、`apt`コマンドによってインストールできる。

「Podmanのインストールコマンド」
```
sudo apt install podman -y
```

インストール後のバージョン確認や、ヘルプを呼び出すコマンドは以下の通り。

「Podmanのバージョン確認コマンド」
```
podman --version
```

「Podmanのヘルプ呼び出しコマンド」
```
podman --help
```

## 各コマンドについて
Podmanで使われるコマンドは、Dockerにおけるコマンドと基本的に一致するように作られている。つまりDockerにおけるコマンドの`docker`である部分を`podman`へと書き換えるだけで良い。

「取得したイメージを一覧」
```
$ sudo podman images
REPOSITORY            TAG     IMAGE ID      CREATED     SIZE
quay.io/podman/hello  latest  df3d2017601b  5 days ago  82.3 kB
```

「テスト用イメージの実行」
```
$ sudo podman run hello
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

## Containerfileを用いたイメージの作成
`Containerfile`というファイルとコマンド:`podman build`を用いてイメージを作成できる。この`Containerfile`はDockerにおける`Dockerfile`に相当し、土台となるイメージの指定やコンテナとして起動するときに実行するコマンドなどを記述しておける。以下に具体的な使用法をコマンドを交えて説明する。

「Containerfileの作成コマンド」
```
touch ./Containerfile
```

「Containerfileをvimで編集するコマンド」
```
vim Containerfile
```

記述内容も`Dockerfile`と同様で問題ないようだ。`FROM`の項に使用するイメージを指定する。

「Containerfileの内容」
```
# Containerfile
FROM debian:bullseye-slim
```

「buildコマンドの実行結果」
```
$ sudo podman build ./
STEP 1: FROM debian:bullseye-slim
Resolved "debian" as an alias (/etc/containers/registries.conf.d/shortnames.conf)
Getting image source signatures
Copying blob 3f9582a2cbe7 done  
Copying config 2248554297 done  
Writing manifest to image destination
Storing signatures
STEP 2: COMMIT
--> 22485542975
22485542975431c307c8297759b906714990ddf703411ab9c5aa8a3af1889676
```

これでイメージが作成されているので、imagesコマンドで確認する。

「作成したイメージの確認」
```
$ sudo podman images
REPOSITORY                TAG            IMAGE ID      CREATED      SIZE
quay.io/podman/hello      latest         df3d2017601b  5 days ago   82.3 kB
docker.io/library/debian  bullseye-slim  224855429754  2 weeks ago  84 MB
```

`Containerfile`のFROMの項に記述した`debian:bullseye-slim`が作成されているのがわかる。

## Podについて
`Pod`はコンテナの集合体やグループに相当するようだ。また単位でもあるので、単一のコンテナで構成されていてもPodと呼ぶらしい。まずは`--help`を付加したコマンドの実行結果を見る。

「pod --helpの実行結果」
```
$ sudo podman pod --help
Manage pods

Description:
  Pods are a group of one or more containers sharing the same network, pid and ipc namespaces.

Usage:
  podman pod [command]

Available Commands:
  create      Create a new empty pod
  exists      Check if a pod exists in local storage
  inspect     Displays a pod configuration
  kill        Send the specified signal or SIGKILL to containers in pod
  pause       Pause one or more pods
  prune       Remove all stopped pods and their containers
  ps          List pods
  restart     Restart one or more pods
  rm          Remove one or more pods
  start       Start one or more pods
  stats       Display a live stream of resource usage statistics for the containers in one or more pods
  stop        Stop one or more pods
  top         Display the running processes of containers in a pod
  unpause     Unpause one or more pods
```

### Podの作成
次のコマンドで中身が無い、つまり空のPodが作成される。

「Podを作成するコマンド」
```
sudo podman pod create
```

Podの一覧は`pod ps`コマンドで表示される。

「Podの一覧を表示するコマンド」
```
sudo podman pod ps
```

`pod create`に`-n`オプションを加えるとPodに名付け可能なようだ。なお指定しない場合は自動生成されたと思われる名前になる。

「Podの名称についての試行」
```
$ sudo podman pod create
0a7c8532b5e0fb5064b894cc1381a62b53253dd450ad088ef0822ce37efceb04

$ sudo podman pod create -n test
a7dda5c3afd6fe6d208cd1dc5e95bf30e75cfa071ed7588fc2ff36218c754916

$ sudo podman pod ps
POD ID        NAME                  STATUS   CREATED         INFRA ID      # OF CONTAINERS
a7dda5c3afd6  test                  Created  3 seconds ago   d310f7b443ee  1
0a7c8532b5e0  dreamy_chandrasekhar  Created  34 minutes ago  c8eb89e97020  1
```

Podを作成すると、`k8s.gcr.io/pause`というイメージから自動的に起動するコンテナがあるようだ。

「Pod作成時に自動起動するコンテナ」
```
$ sudo podman container ps -a
CONTAINER ID  IMAGE                 COMMAND  CREATED         STATUS   PORTS   NAMES
14aec3d2f773  k8s.gcr.io/pause:3.2           10 seconds ago  Created          0e0e53ad5179-infra
```

### 作成したPodの詳細
Podについて詳細な情報を得たい時は`pod inspect`コマンドを使用する。

「Podの詳細を取得するコマンド」
```
sudo podman pod inspect <id>
```

`<id>`には詳細を確認したいPodのIDが入る。これは`pod ps`コマンドで表示させられる。

「PodのIDを確認」
```
$ sudo podman pod inspect 0a7c8532b5e0 
{
     "Id": "0a7c8532b5e0fb5064b894cc1381a62b53253dd450ad088ef0822ce37efceb04",
     "Name": "dreamy_chandrasekhar",
     "Created": "2023-03-21T00:40:02.410556286+09:00",
     "CreateCommand": [
          "podman",
          "pod",
          "create"
     ],
     "State": "Created",
     "Hostname": "dreamy_chandrasekhar",
     "CreateCgroup": true,
     "CgroupParent": "machine.slice",
     "CgroupPath": "machine.slice/machine-libpod_pod_0a7c8532b5e0fb5064b894cc1381a62b53253dd450ad088ef0822ce37efceb04.slice",
     "CreateInfra": true,
     "InfraContainerID": "c8eb89e97020858cda3a86bce68dc2a2e33f4aca26f6bac88d0feaa4b4a50f04",
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
          "uts",
          "ipc",
          "net"
     ],
     "NumContainers": 1,
     "Containers": [
          {
               "Id": "c8eb89e97020858cda3a86bce68dc2a2e33f4aca26f6bac88d0feaa4b4a50f04",
               "Name": "0a7c8532b5e0-infra",
               "State": "configured"
          }
     ]
}
```

### Podの中でコンテナを起動
Podにコンテナを含めるには`podman pod`コマンドからではなく、`podman run`コマンドにオプションを付けようだ。以下に具体的な手順を記述していく。

まずPodのname(名前)を確認する。

「Podの名称を表示」
```
$ sudo podman pod ps
POD ID        NAME      STATUS   CREATED        INFRA ID      # OF CONTAINERS
0e0e53ad5179  test-pod  Created  6 minutes ago  14aec3d2f773  1
```

次に対象にしたいPodのnameを`--pod`オプションの引数とする。

「名称を引数に指定したコマンド」
```
sudo podman run --pod test-pod --name debian debian:bull
seye-slim
```

これでPodの中に起動したコンテナが含まれている状態になる。

「Podにコンテナが含まれているかの確認」
```
$ sudo podman pod inspect 0e0e53ad5179
{
     "Id": "0e0e53ad517930a994049e6d418686d1e2ce9a58077ec7168892bb9358627fdb",
     "Name": "test-pod",
     "Created": "2023-03-23T22:48:07.118729645+09:00",

     -省略-

     "NumContainers": 2,
     "Containers": [
          {
               "Id": "14aec3d2f773d8c6aff333a64e00d4787de0b744b34d0db3609ff775a5d9f8d2",
               "Name": "0e0e53ad5179-infra",
               "State": "running"
          },
          {
               "Id": "5b4be4aff9b0f0c6e442ebe5c6f09d5d873c5c163e69c2067aafe0e5c2efb750",
               "Name": "debian",
               "State": "exited"
          }
     ]
}
```

### Podから指定のコンテナを取り除く
Podの中に含まれているコンテナは`podman rm`コマンドだけで取り除ける。起動時と違ってオプションなどは必要なかった。

「Podから特定のコンテナを除去」
```
$ sudo podman container rm 5b4be4aff9b0f0c6e442ebe5c6f09d5
d873c5c163e69c2067aafe0e5c2efb750
5b4be4aff9b0f0c6e442ebe5c6f09d5d873c5c163e69c2067aafe0e5c2efb750

$ sudo podman pod inspect 0e0e53ad5179
{
     "Id": "0e0e53ad517930a994049e6d418686d1e2ce9a58077ec7168892bb9358627fdb",
     "Name": "test-pod",
     "Created": "2023-03-23T22:48:07.118729645+09:00",

     -省略-

     "NumContainers": 2,
     "Containers": [
          {
               "Id": "14aec3d2f773d8c6aff333a64e00d4787de0b744b34d0db3609ff775a5d9f8d2",
               "Name": "0e0e53ad5179-infra",
               "State": "running"
          }
     ]
}
```

### Podに接続
Podと通信するには、Podを作成する際に`-p`オプションでポートを割り当てる

「Podへのポートの割り当てコマンド」
```
sudo podman pod create --name test-pod -p 9001:80
```
`-p`オプションでは、コロン記号の左側に使用している端末側に...つまり実際に使用するポート番号を、右側にPodやコンテナが使おうとするポート番号を記述する。この例では9001番ポートにアクセスするとPodの80番ポートにアクセスする事と同等になる。

`podman pod inspect`コマンドでポート割り当ての様子を見ることも出来る。

「Podのポート番号を確認」
```
$ sudo podman pod inspect test-pod
{
     "Id": "188f0517ba031309629c0592d00fd847c0a48980233d10ec9a5449de764b07ca",
     "Name": "test-pod",
     "Created": "2023-03-26T01:14:13.50034823+09:00",
     "CreateCommand": [
          "podman",
          "pod",
          "create",
          "--name",
          "test-pod",
          "-p",
          "9001:80"
     ],
     "State": "Running",
     "Hostname": "test-pod",
     "CreateCgroup": true,
     "CgroupParent": "machine.slice",
     "CgroupPath": "machine.slice/machine-libpod_pod_188f0517ba031309629c0592d00fd847c0a48980233d10ec9a5449de764b07ca.slice",
     "CreateInfra": true,
     "InfraContainerID": "dc30103f70b9ab2a56818470efee40358213f2a122a66d93b6426ddc5017f45d",
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
     -省略-
}
```

ポート割り当ての動作確認としては`curl`コマンドを用いればよい。NginxなどのWebサーバに関してはWebブラウザを使う手もある。

「curlのよるポート割り当ての確認」
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

## 参考文献
1. https://access.redhat.com/documentation/ja-jp/red_hat_enterprise_linux/8/html-single/building_running_and_managing_containers/index#proc_building-a-container-inside-a-podman-container_assembly_running-skopeo-buildah-and-podman-in-a-container　(18.9. Podman コンテナー内でのコンテナーのビルド Red Hat Enterprise Linux 8 | Red Hat Customer Portal)
2. https://www.redhat.com/ja/topics/containers/what-is-kubernetes-pod (Kubernetes Pod とは | Red Hat)
3. https://kubernetes.io/ja/docs/concepts/workloads/pods/ (Pod | Kubernetes)
