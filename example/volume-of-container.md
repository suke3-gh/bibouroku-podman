# bibouroku-podman > volume-of-container

文書作成日:2023年04月11日 最終更新日:2023年07月03日

コンテナのボリュームを設定し、コンテナと端末側でファイルなどを共有する手順について記述する。

1. [使用するイメージの取得](#使用するイメージの取得)
2. [使用するPodの生成](#使用するpodの生成)
3. [各コンテナの起動](#各コンテナの起動)
    1. [Nginx](#nginx)
    2. [PHP](#php)
4. [端末側のファイルをコンテナ側に反映](#端末側のファイルをコンテナ側に反映)
5. [動作環境](#動作環境)
6. [参考文献](#参考文献)

## 使用するイメージの取得
WebサーバのNginxと、そこで作動するPHPのイメージを使用する。イメージを取得するためのコマンドは以下の通り。

「Nginxイメージの取得コマンド」
```
sudo podman pull nginx:1.23.4-alpine3.17
```

「PHPイメージの取得コマンド」
```
sudo podman pull php:8.2-fpm-alpine3.17
```

## 使用するPodの生成
利便性と.yamlフォーマットをより理解するためにPodを使用する。Podの80番ポートを端末の9001番ポートに割り当て、名称を「VolumeTest」にしたPodを次のコマンドで生成する。

「Podの作成コマンド」
```
sudo podman pod create -p 9001:80 --name VolumeTest
```

## 各コンテナの起動
ボリュームの設定は`podman run`コマンドを実行する時に、オプションとして行う。NginxとPHPについて個別に設定していく。

### Nginx
まずコンテナ内に存在する`default.conf`というNginxの設定ファイルを入手するために、一度Nginxコンテナを起動する。

「Nginxコンテナの起動」
```
sudo podman run -dt --name nginx nginx:
1.23.4-alpine3.17
```

次にNginxコンテナの中に入り、設定ファイルのパスを確認する。使用しているイメージがAlpineベースなので、`podman exec`コマンドで用いるものは`bash`ではなく`ash`となる。

「Nginxコンテナ内へ入るコマンド」
```
sudo podman exec -it nginx ash
```

コンテナ内で`nginx`コマンドを`-t`オプション付きで実行してパスを表示させる。

「Nginx関係のファイルパスを表示」
```
/ # nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

`/etc/nginx/`に設定ファイルが存在するので、ここの`conf.d`ディレクトリに目的のファイルが存在する。

「default.confの内容を表示」
```
/ # cd /etc/nginx/conf.d/

/etc/nginx/conf.d # ls
default.conf

/etc/nginx/conf.d # cat default.conf
server {
    listen       80;
    listen  [::]:80;
    server_name  localhost;

    #access_log  /var/log/nginx/host.access.log  main;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    # proxy the PHP scripts to Apache listening on 127.0.0.1:80
    #
    #location ~ \.php$ {
    #    proxy_pass   http://127.0.0.1;
    #}

    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #
    #location ~ \.php$ {
    #    root           html;
    #    fastcgi_pass   127.0.0.1:9000;
    #    fastcgi_index  index.php;
    #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
    #    include        fastcgi_params;
    #}

    # deny access to .htaccess files, if Apache's document root
    # concurs with nginx's one
    #
    #location ~ /\.ht {
    #    deny  all;
    #}
}
```

`podman cp`コマンドを用いて、コンテナ内のファイルを端末側にコピーする。第一句はコピー元になる対象、つまりコンテナ内のdefault.confファイル。第二句はコピー先、ここでは上記で作成したディレクトリを指定する。

「default.confを端末側にコピーするコマンド」
```
sudo podman cp nginx:/etc/nginx/conf.d/default.conf ./nginx/conf.d/default.conf
```

目的ファイルを得たので、一度Nginxコンテナを取り除く。`-f`オプションを付加すると、コンテナを停止を行わずにそのまま削除出来る。

「Nginxコンテナを削除するコマンド」
```
sudo podman container rm -f nginx
```

次に、コンテナと共有される端末側のディレクトリを用意する。上記の`default.conf`のためと、Webサーバのrootとなるディレクトリを作成する。複数層のディレクトリをまとめて作成するので`-p`オプションは必須。

「default.confを配置するディレクトリを作成するコマンド」
```
mkdir -p ./nginx/conf.d/
```

「Webサーバのrootとなるディレクトリを作成するコマンド」
```
mkdir -p ./nginx/html/
```

再びコンテナを起動するが、ここでディレクトリやファイルを共有するためのボリュームを`-v`オプションで指定する。`-v`オプションの値はコロン記号の左側に端末側ボリュームを、右側にコンテナ側のボリュームを記述する。コンテナにマウントされるので、端末側がコンテナ側を上書きするという挙動のようだ。

「Nginxコンテナを起動するコマンド」
```
sudo podman run -dt -v ./nginx/conf.d/:/etc/nginx/conf.d/ -v ./nginx/html/:/usr/share/nginx/html/ --name nginx --pod VolumeTest nginx:1.23.4-alpine3.17
```

### PHP
Nginxの場合と同様に、まずコンテナを起動する。

「PHPコンテナを起動するコマンド」
```
sudo podman run -it --name php php:8.2-fpm-alpine3.17
```

`php.ini`というファイルが必要なので起動したPHPコンテナの中に入り、そのパスを確認する。使用しているイメージがAlpineベースなので、`podman exec`コマンドで用いるものは`bash`ではなく`ash`となる。

「PHPコンテナの中に入るコマンド」
```
sudo podman exec -it php ash
```

コンテナ内で`php --ini`コマンドを実行することでパスが表示される。

「php.iniのパスの確認」
```
/var/www/html # php --ini
Configuration File (php.ini) Path: /usr/local/etc/php
Loaded Configuration File:         (none)
Scan for additional .ini files in: /usr/local/etc/php/conf.d
Additional .ini files parsed:      /usr/local/etc/php/conf.d/docker-fpm.ini,
/usr/local/etc/php/conf.d/docker-php-ext-sodium.ini 

/var/www/html # cd /usr/local/etc/php/
/usr/local/etc/php # ls -a
.                    conf.d               php.ini-production
..                   php.ini-development
```

`php.ini-production`を端末側にコピーするので、端末側でこのファイルを配置するディレクトリを作成する。

「php.iniを配置するディレクトリを作成するコマンド」
```
mkdir -p ./php/
```

次に`podman cp`コマンドを使う。ここで`php.ini-production`を`php.ini`という名称に変更している。

「php.iniを端末側にコピーするコマンド」
```
sudo podman cp php:/usr/local/etc/php/php.ini-productio
n ./php/php.ini
```

「php.iniの確認」
```
$ cat ./usr/local/etc/php/php.ini
[PHP]

;;;;;;;;;;;;;;;;;;;
; About php.ini   ;
;;;;;;;;;;;;;;;;;;;
; PHP's initialization file, generally called php.ini, is responsible for
; configuring many of the aspects of PHP's behavior.

; PHP attempts to find and load this configuration from a number of locations.
; The following is a summary of its search order:
; 1. SAPI module specific location.
; 2. The PHPRC environment variable.
; 3. A number of predefined registry keys on Windows
; 4. Current working directory (except CLI)
; 5. The web server's directory (for SAPI modules), or directory of PHP
; (otherwise in Windows)
; 6. The directory from the --with-config-file-path compile time option, or the
; Windows directory (usually C:\windows)
; See the PHP docs for more specific information.
; https://php.net/configuration.file\
-以下省略-
```

ファイルの用意が完了したので、起動していたコンテナの削除を行う。

「PHPコンテナを削除するコマンド」
```
sudo podman container rm -f php
```

php.iniをボリュームに指定してコンテナを起動する。実際にPHPを動作させるにはNginxのルートディレクトリもボリュームに含める必要があることに注意。

「PHPコンテナを起動するコマンド」
```
sudo podman run -dt -v ./php/php.ini:/usr/local/etc/php/php.ini -v ./nginx/html/:/usr/share/nginx/html/ --name php --pod VolumeTest php:8.2-fpm-alpine3.17
```

## 端末側のファイルをコンテナ側に反映
`./nginx/html/`に`page.html`というファイルを作成する。このディレクトリはNginxコンテナと共有されているので、`localhost:9001/page.html`に接続するとpage.htmlが表示されることになる。

「page.htmlを作成するコマンド」
```
touch -p ./nginx/html/page.html
```

「vimでpage.htmlを編集するコマンド」
```
vim ./nginx/html/page.html
```

「page.htmlの内容を表示する」
```
$ cat ./nginx/html/page.html
test message in page.html
```

「curlでpage.htmlにアクセスして動作確認」
```
$ curl localhost:9001/page.html
test message in page.html
```

これで端末側のファイルをコンテナ側に反映出来たことになる。端末側のdefault.confの変更をコンテナに反映するには、`podman container restart`コマンドを使用してコンテナの再起動を行う。PHP.iniについても同様。

## 動作環境
「podman infoの実行結果」
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
  memFree: 2511826944
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
  uptime: 735h 11m 23.95s (Approximately 30.62 days)
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
1. https://hub.docker.com/_/nginx (nginx - Official Image | Docker Hub)
2. https://hub.docker.com/_/php (php - Official Image | Docker Hub)
