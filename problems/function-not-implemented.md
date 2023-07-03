# bibouroku-podman > function-not-implemented

文書作成日:2023年03月18日 最終更新日:2023年06月30日

Podmanをインストール後にコマンド: `podman run`でコンテナの起動を試みたところ、コンテナは作成されたが以下のようなエラーで起動しない状態に陥った。これを解決するために行った事を記す。

「コンテナの実行エラー」
```
$ sudo podman run debian
Resolved "debian" as an alias (/etc/containers/registries.conf.d/shortnames.conf)
Trying to pull docker.io/library/debian:latest...
Getting image source signatures
Copying blob 1e4aec178e08 done  
Copying config 54e726b437 done  
Writing manifest to image destination
Storing signatures
Error: OCI runtime error: create keyring `24632dc3c53f557818bb8d4d4a23f09e0d9dd7be9b608ae8a9f82d02b7477a90`: Function not implemented
```

1. [対処1:開発版の使用](#対処1開発版の使用)
    1. [kubic project pageからの入手](#kubic-project-pageからの入手)
    2. [要求されたcriuのインストール](#要求されたcriuのインストール)
    3. [criuに対するdpkgコマンドの実行](#criuに対するdpkgコマンドの実行)
    4. [criuに対するapt showコマンドの実行](#criuに対するapt-showコマンドの実行)
    5. [criuのパッケージ依存関係の確認](#criuのパッケージ依存関係の確認)
    6. [Linux環境の初期化](#linux環境の初期化)
    7. [aptキャッシュの削除](#aptキャッシュの削除)
    8. [criuのパッケージを入手してローカルでインストール](#criuのパッケージを入手してローカルでインストール)
    9. [libgpgme.so.11の不足](#libgpgmeso11の不足)
    10. [glibc_2.33の不足](#glibc_233の不足)
    11. [断念:対処1についての結論](#断念対処1についての結論)
2. [対処2: ランタイムの変更](#対処2-ランタイムの変更)
    1. [runcのインストール](#runcのインストール)
    2. [containers.confの編集](#containersconfの編集)
    3. [解決:対処2についての結論](#解決対処2についての結論)
3. [動作環境](#動作環境)
4. [参考文献](#参考文献)

## 対処1:開発版の使用
これについて検索した結果、同様と思われる状況についての質問を発見。Chromebook上でのLinux環境というところまで一致している。これは出発点として解決を目指したが、結果は「断念」した。

- 発見したIssue: https://github.com/containers/podman/issues/6818 (created pods are not startable · Issue #6818 · containers/podman)

### Kubic project pageからの入手
まず`Kubic project page`からの入手する`Podman`の使用を試す。Podman公式ドキュメントの記述に従い、以下のコマンドを実行。

「ディレクトリを作成するコマンド」
```
sudo mkdir -p /etc/apt/keyrings
```

「取得先リポジトリの設定を行うコマンド」
```
curl -fsSL https://download.opensuse.org/repositories/devel:kubic:libcontainers:unstable/Debian_Testing/Release.key \
  | gpg --dearmor \
  | sudo tee /etc/apt/keyrings/devel_kubic_libcontainers_unstable.gpg > /dev/null
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/devel_kubic_libcontainers_unstable.gpg]\
    https://download.opensuse.org/repositories/devel:kubic:libcontainers:unstable/Debian_Testing/ /" \
  | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:unstable.list > /dev/null
```

Podmanのパッケージを入手しようとしたところ、以下のエラーが生じた。

「Podmanの取得とエラーメッセージ」
```
$ sudo apt-get update
$ sudo apt-get -y upgrade
$ sudo apt-get -y install podman
Some packages could not be installed. This may mean that you have
requested an impossible situation or if you are using the unstable
distribution that some required packages have not yet been created
or been moved out of Incoming.
The following information may help to resolve the situation:

The following packages have unmet dependencies:
 crun : Depends: criu but it is not installable
E: Unable to correct problems, you have held broken packages.
```

### 要求されたcriuのインストール
開発版インストール時のエラーメッセージによると、`criu`が必要だがインストール不可であるようなので、まず`apt`コマンドによってパッケージのインストールを試す。しかしこれもエラーが生じた。

「crunパッケージ取得時のエラーメッセージ」
```
$ sudo apt install criu
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
Package criu is not available, but is referred to by another package.
This may mean that the package is missing, has been obsoleted, or
is only available from another source

E: Package 'criu' has no installation candidate
```

### criuに対するdpkgコマンドの実行
今度は`criu`というパッケージの候補が存在しないというエラーが生じたので、一度criuについて`dpkg`コマンドの実行結果を以下に示しておく。

「dpkgコマンドの実行結果」
```
$ dpkg -l criu
dpkg-query: no packages found matching criu

$ dpkg -s criu
dpkg-query: package 'criu' is not installed and no information is available

$ dpkg -i criu
dpkg: error: failed to read archive 'criu': No such file or directory
```
`criu`というパッケージはインストールされていない扱いのようだ。

### criuに対するapt showコマンドの実行
次に`apt show`コマンドでcriuパッケージの詳細を確認したところパッケージ自体が見つからなかった。

「apt showコマンドの実行結果」
```
$ apt show criu
Package: criu
State: not a real package (virtual)
N: Can't select candidate version from package criu as it has no candidate
N: Can't select versions from package 'criu' as it is purely virtual
N: No packages found
```

### criuのパッケージ依存関係の確認
criuパッケージの依存関係も確認した。

「apt-cache showpkgのコマンド実行結果」
```
$ apt-cache showpkg criu
Package: criu
Versions: 

Reverse Depends: 
  runc,criu
  crun,criu
Dependencies: 
Provides: 
Reverse Provides: 
```

### Linux環境の初期化
ここでChromeBookのLinux開発環境を再構築した。改めてPodmanをインストールしてみたが、`OCI runtime error`は相変わらずで状況は変化しなかった。

上記のパッケージ依存関係までの手順を再び行ったが結果はすべて変化なし。

### aptキャッシュの削除
パッケージマネージャのaptのキャッシュ削除や修復を試みる。

「aptのキャッシュ削除コマンド」
```
sudo apt clean
```

「apt updateの実行コマンド」
```
sudo apt update
```

「-fオプションによる修復コマンド」
```
sudo apt -f install
```

上記のコマンドを実行後に`criu`のインストールを行ったがエラーメッセージに変化なし。

### criuのパッケージを入手してローカルでインストール
criuのdebファイルを直接入手し、Chromebookの機能でインストールすることを試す。

- 入手先: https://mirrorcache-jp.opensuse.org/repositories/devel:/tools:/criu/Debian_11/amd64/ (/repositories/devel:/tools:/criu/Debian_11/amd64 - openSUSE Download)

「dpkgコマンドの実行結果」
```
$ dpkg -s criu
Package: criu
Status: install ok installed
Priority: optional
Section: admin
Installed-Size: 3119
Maintainer: Salvatore Bonaccorso <carnil@debian.org>
Architecture: amd64
Version: 3.17.1-1
Depends: libgnutls30 (>= 3.7.0), python3-protobuf, python3:any, libbsd0 (>= 0.6.0), libc6 (>= 2.28), libnet1 (>= 1.1.2.1), libnl-3-200 (>= 3.2.7), libprotobuf-c1 (>= 1.0.1)
Recommends: iproute2 | iproute
Description: checkpoint and restore in userspace
 criu contains the utilities to checkpoint and restore processes in
 userspace.
Homepage: https://www.criu.org/
```

この状態でaptコマンドによるpodmanのインストールに成功した。

### libgpgme.so.11の不足
podmanコマンドを実行したところ以下のエラーが発生した。

「libgpgme.so.11の不足によるエラーメッセージ」
```
$ podman
podman: error while loading shared libraries: libgpgme.so.11: cannot open shared object file: No such file or directory
```

`libgpgme.so.11`というファイルが足りないようだ。このファイルは`libgpgme11`パッケージに含まれているようなのでこれをインストールする。
```
sudo apt install libgpgme11
```

- パッケージ情報: https://packages.debian.org/bullseye/libgpgme11 (Debian -- bullseye の libgpgme11 パッケージに関する詳細)

### GLIBC_2.33の不足
再びpodmanコマンドを実行すると別の問題が発生した。

「GLIBC_2.33の不足によるエラーメッセージ」
```
$ podman
podman: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.33' not found (required by podman)
podman: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.32' not found (required by podman)
podman: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.34' not found (required by podman)
```

`GNU C`など共有ライブラリに不足があるようなので、それに関わる情報を表示する`ldd`コマンドを実行する。

「lddコマンドの実行結果」
```
$ ldd --version
ldd (Debian GLIBC 2.31-13+deb11u5) 2.31
Copyright (C) 2020 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
Written by Roland McGrath and Ulrich Drepper.
```

2023/02/28時点でインストールされている`GLIBC`のバージョンが2.31であり、要求されているバージョンが2.32以上であることがこの問題の原因のようだ。

要求されたバージョンの`glibc`を自前で用意することにした。

以下のコマンドで`.deb`ファイルをダウンロードする。

「libcのdebファイル入手コマンド」
```
wget http://ftp.jp.debian.org/debian/pool/main/g/glibc/libc6_2.36-8_amd64.deb
```
`wget`は指定したURLからダウンロードを行うダウンローダーであり、対象を現在のディレクトリに保存する。

ダウンロードした`.deb`ファイルを`apt`コマンドでインストールするが、ここでもエラーが生じた。

「libcのインストールコマンド」
```
sudo apt install ./libc6_2.36-8_amd64.deb
```

「libcインストールでのエラーメッセージ」
```
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
Note, selecting 'libc6' instead of './libc6_2.36-8_amd64.deb'
The following package was automatically installed and is no longer required:
  libslirp0
Use 'sudo apt autoremove' to remove it.
Suggested packages:
  glibc-doc locales libnss-nis libnss-nisplus
The following packages will be REMOVED:
  libc-bin locales
The following packages will be upgraded:
  libc6
WARNING: The following essential packages will be removed.
This should NOT be done unless you know exactly what you are doing!
  libc-bin
1 upgraded, 0 newly installed, 2 to remove and 9 not upgraded.
Need to get 0 B/2,747 kB of archives.
After this operation, 20.1 MB disk space will be freed.
You are about to do something potentially harmful.
To continue type in the phrase 'Yes, do as I say!'
 ?] Yes, do as I say!               
Get:1 /home/suke3/workspace/sunaba/podman/libc6_2.36-8_amd64.deb libc6 amd64 2.36-8 [2,747 kB]
Preconfiguring packages ... 
(Reading database ... 37423 files and directories currently installed.)
Removing locales (2.31-13+deb11u5) ...
dpkg: warning: overriding problem because --force enabled:
dpkg: warning: this is an essential package; it should not be removed
Removing libc-bin (2.31-13+deb11u5) ...
dpkg: warning: 'ldconfig' not found in PATH or not executable
dpkg: error: 1 expected program not found in PATH or not executable
Note: root's PATH should usually contain /usr/local/sbin, /usr/sbin and /sbin
E: Sub-process /usr/bin/dpkg returned an error code (2)
```

`libc-bin`が削除されたことによって`ldconfig`にある共有ライブラリへのパスが見つからなくなっているようだ。
これについては解決策がみつかった。

- 情報元: https://www.thetqweb.com/forum/wsl/fix-dpkg-warning-ldconfig-not-found-in-path-or-not-executable-in-wsl/ (Fix "dpkg: warning: 'ldconfig' not found in PATH or not executable" in WSL | thetqweb)

まず`libc-bin`のパッケージを入手する。
- 入手先: https://packages.debian.org/sid/amd64/libc-bin/download (Debian -- パッケージのダウンロードに関する選択 -- libc-bin_2.36-9_amd64.deb)

次にダウンロードしたパッケージの中身を抽出し、目的のファイルをシステム側にコピーする。

「パッケージの内容を抽出するコマンド」
```
sudo dpkg --extract ./libc-bin_2.36-8_amd64.deb ./
```

「抽出したファイルのコピーするコマンド」
```
sudo cp ./sbin/ldconfig /sbin/
```

そして`locales`と`libc-bin`の再インストールを行う。

「localesのインストールコマンド」
```
sudo apt install --reinstall locales
```

「libc-binのインストールコマンド」
```
sudo apt install --reinstall libc-bin
```

一方で`ibc-bin_2.36-8`を直接インストールしようとしても必要なバージョンの`libc6`がインストールされていないとされて失敗する。

「libc6の不足によるエラーメッセージ」
```
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
Note, selecting 'libc-bin' instead of './libc-bin_2.36-8_amd64.deb'
Some packages could not be installed. This may mean that you have
requested an impossible situation or if you are using the unstable
distribution that some required packages have not yet been created
or been moved out of Incoming.
The following information may help to resolve the situation:

The following packages have unmet dependencies:
 libc-bin : Depends: libc6 (> 2.36) but 2.31-13+deb11u5 is to be installed
            Recommends: manpages but it is not going to be installed
E: Unable to correct problems, you have held broken packages.
```

### 断念:対処1についての結論
結果:「この手段での解決を断念」

`glibc`はLinuxのカーネルと密接に関わっているので単体での更新が困難のようだ。また再びLinux開発環境を初期化する。以下にaptコマンドによるリポジトリの参照先を元に戻る手段を記しておく。

`apt`コマンドで参照するリポジトリの情報は`/etc/apt/sources.list.d`に存在しているようなので、そのディレクトリに移動して`list`ファイルを削除する。gpgキーも同様に削除する。

「listファイルがあるディレクトリへ移動するコマンド」
```
cd /etc/apt/sources.list.d
```

「listファイルの削除コマンド」
```
sudo rm devel:kubic:libcontainers:unstable.list
```

「gpgキーがあるディレクトリへ移動するコマンド」
```
cd /etc/apt/keyrings
```

「gpgキーの削除コマンド」
```
sudo rm devel_kubic_libcontainers_unstable.gpg
```

これによってDebianの公式リポジトリからパッケージがダウンロードされるようになる。

## 対処2: ランタイムの変更
ランタイムに関するエラーであるので、使用するランタイムの変更を試みる。

Podmanで要求されるOCIランタイムは`crun`あるいは`runc`であり、初期状態では前者がインストールされていて、Podmanもその前者を指定している。よって変更先として`runc`を試してみることにする。

runcについて、`apt show`コマンドを実行した結果を記しておく。

「runcに対するapt showコマンドの実行結果」
```
$ apt show runc
Package: runc
Version: 1.0.0~rc93+ds1-5+deb11u2
Built-Using: go-md2man-v2 (= 2.0.0+ds-5), golang-1.15 (= 1.15.15-1~deb11u4), golang-blackfriday-v2 (= 2.0.1-3), golang-dbus (= 5.0.3-2), golang-github-checkpoint-restore-go-criu (= 4.1.0-3), golang-github-cilium-ebpf (= 0.2.0-1), golang-github-containerd-console (= 1.0.1-2), golang-github-coreos-go-systemd (= 22.1.0-3), golang-github-cyphar-filepath-securejoin (= 0.2.2-2), golang-github-docker-go-units (= 0.4.0-3), golang-github-moby-sys (= 0.0~git20201113.5a29239-1), golang-github-mrunalp-fileutils (= 0.5.0-1), golang-github-opencontainers-selinux (= 1.8.0-1), golang-github-opencontainers-specs (= 1.0.2.41.g7413a7f-1+deb11u1), golang-github-pkg-errors (= 0.9.1-1), golang-github-shurcool-sanitized-anchor-name (= 1.0.0-1), golang-github-vishvananda-netlink (= 1.1.0-2), golang-github-vishvananda-netns (= 0.0~git20200728.db3c7e5-1), golang-github-willf-bitset (= 1.1.11-1), golang-gocapability-dev (= 0.0+git20200815.42c35b4-1), golang-golang-x-sys (= 0.0~git20210124.22da62e-1), golang-goprotobuf (= 1.3.4-2), golang-logrus (= 1.7.0-2)
Priority: optional
Section: devel
Maintainer: Debian Go Packaging Team <team+pkg-go@tracker.debian.org>
Installed-Size: 8,404 kB
Depends: libc6 (>= 2.14), libseccomp2 (>= 2.4.1)
Recommends: criu
Breaks: docker.io (<= 1.13.1~ds1-2)
Homepage: https://github.com/opencontainers/runc
Download-Size: 2,428 kB
APT-Sources: https://deb.debian.org/debian bullseye/main amd64 Packages
Description: Open Container Project - runtime
 "runc" is a command line client for running applications packaged according
 to the Open Container Format (OCF) and is a compliant implementation of
 the Open Container Project specification.
```

### runcのインストール
「runcのインストールコマンド」
```
sudo apt install runc
```

### containers.confの編集
使用するランタイムを指定するには`/etc/containers/`にある`containers.conf`というファイルに記述が必要なようだ。よってこのファイルを次のコマンドで作成し、作成したファイルを`vim`で編集する。

「container.confの作成コマンド」
```
sudo touch /etc/containers/containers.conf
```

「container.confをvimで編集するコマンド」
```
sudo vim /etc/containers/containers.conf
```

「container.confに記述する内容」
```
[engine]
runtime = "runc"
```

### 解決:対処2についての結論
結果:「解決」

ランタイムを`runc`に指定した後にコマンド: `$ sudo podman run debian`を実行してもエラーが発生しなかった。コンテナは起動してすぐ停止したようだ。

「コンテナの様子」
```
$ sudo podman ps -l
CONTAINER ID  IMAGE                            COMMAND  CREATED        STATUS                    PORTS   NAMES
6375efc57734  docker.io/library/debian:latest  bash     3 seconds ago  Exited (0) 3 seconds ago          cranky_heyrovsky
```

Podmanのセットアップが完了しているかを確かめるイメージも無事に実行されたので、この問題は解決したと判断する。

「helloイメージの実行結果」
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

## 参考文献
1. https://access.redhat.com/documentation/ja-jp/red_hat_enterprise_linux/8/html-single/building_running_and_managing_containers/index#selecting-a-container-runtime_building-running-and-managing-containers (コンテナーの構築、実行、および管理 Red Hat Enterprise Linux 8 | Red Hat Customer Portal)
