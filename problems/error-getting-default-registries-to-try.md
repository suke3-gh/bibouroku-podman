# bibouroku-podman > error-getting-default-registries-to-try

文書作成日:2023年03月18日 最終更新日:2023年6月30日

Linux開発環境の初期化を実行した後に、Podmanをインストールした。動作確認のために以下のコマンドを実行したが、エラーが発生した。この問題を解決するまでの過程を記述する。

「httpdイメージの取得でのエラーメッセージ」
```
$　sudo podman pull httpd
Error: error getting default registries to try: short-name "httpd" did not resolve to an alias and no unqualified-search registries are defined in "/etc/containers/registries.conf"
```

1. [レジストリの指定](#レジストリの指定)
2. [registries.confの編集](#registriesconfの編集)
3. [結果と結論](#結果と結論)
4. [動作環境](#動作環境)
5. [参考文献](#参考文献)

## レジストリの指定
エラーメッセージの内容を確認すると、`/etc/containers/`にある`registries.conf`というファイルにおいて何らかの定義...つまり記述が足りないようだ。これはイメージの入手先であるレジストリが適切に指定されていないときに生じるエラーであるそうなので、それを補う記述を追加することで解決を図る。

## registries.confの編集
ディレクトリ:`/etc/containers/registries.conf.d/`の下に別ファイルを用意することで`registries.conf`を直接編集することなく設定を追加できる。よって、次の二つのコマンドを使用して追加のファイルを作成する。

「ファイルの作成コマンド1」
```
sudo touch /etc/containers/registries.conf.d/00-unqualified-search-registries.conf
```

「ファイルの作成コマンド2」
```
sudo touch /etc/containers/registries.conf.d/01-registries.conf
```

各ファイルの内容は以下の通り。

「00-unqualified-search-registries.confの内容」
```
unqualified-search-registries = ["quay.io', 'docker.io"]
```

「01-registries.confの内容」
```
[[registry]]
location = "docker.io"
```

## 結果と結論
結果:「解決」

上記の手順を行った後、コマンド:`$ sudo podman pull httpd`を実行すると、エラーが発生することなくイメージの入手に成功した。
Dockerを予めインストールしていると、このエラーが生じないようだ。

## 動作環境
|項目|内容|
|-|-|
|ハードウェア|ASUS C425TA|
|OS|Google ChromeOS 110.0.5481.181（Official Build） (64 ビット)|
|Podman|version 3.0.1|

「OSのバージョン情報」
```
$ cat /proc/version
Linux version 5.10.159-20950-g3963226d9eb4 (chrome-bot@chromeos-release-builder-us-central1-b-x32-6-h80v) (Chromium OS 15.0_pre465103_p20220825-r13 clang version 15.0.0 (/var/tmp/portage/sys-devel/llvm-15.0_pre465103_p20220825-r13/work/llvm-15.0_pre465103_p20220825/clang db1978b67431ca3462ad8935bf662c15750b8252), LLD 15.0.0) #1 SMP PREEMPT Mon Feb 20 18:10:10 PST 2023
```

## 参考文献
1. https://wiki.archlinux.jp/index.php/Podman (Podman - ArchWiki)
2. https://zenn.dev/dozo/articles/0ced3feae9ac63 (WSL2にDocker代替のPodmanを入れてみる: ポッドマンが倒せない(1))
3. https://docs.oracle.com/cd/F61410_01/podman/podman-UsingContainerRegistries.html#podman-registry (Podmanユーザー・ガイド)
