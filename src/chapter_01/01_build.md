# 1. ビルドと実行

ビルドと実行は、`make`と`make qemu`で行えるとREADMEに記載がある。
また以下の記事から`make qemu-nox`でXなしで実行でき、`make qemu-nox-gdb`でXなしでgdbでデバッグできることが分かる。
[「xv6のデバッグ環境をつくる」（リンク2）](https://qiita.com/ksky/items/974ad1249cfb2dcf5437)

コンパイル時にレベル2の最適化オプションが付与されているので、デバッグの際に変数の中身が見れないことがある。
そのときは、MakefileのCFLAGSからオプションO2を削ると変数の中身が見える。

Makefile
```Makefile
#CFLAGS = -fno-pic -static -fno-builtin -fno-strict-aliasing -O2 -Wall -MD -ggdb -m32 -Werror -fno-omit-frame-pointer
CFLAGS = -fno-pic -static -fno-builtin -fno-strict-aliasing -Wall -MD -ggdb -m32 -fno-omit-frame-pointer
```

シングルスレッドでの動作を確認したいときは、MakefileのCPUS変数の値を1にする。

Makefile
```Makefile
#CPUS := 2
CPUS := 1
```

また、wsl上でqemuやgdbを動かしていて、キーボード入力などのためにXを使いたいとき（qemu-noxで実行するとシリアルポートを使うことになる）は、windows側でXサーバを起動し、wsl側の環境変数DISPLAYを`<win側のvethのアドレス>:0`として、`make qemu-gdb`を実行する。
XサーバにVcXsrvを使う場合は以下の記事が参考になる。
[「AsTechLog WSL2＋Ubuntu 20.04でGUIアプリを動かす」（リンク26）](https://astherier.com/blog/2020/08/run-gui-apps-on-wsl2/#toc_id_2)
