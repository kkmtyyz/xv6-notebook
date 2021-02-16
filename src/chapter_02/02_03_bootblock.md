# 2.3. ターゲットbootblock
bootblockは次のように作られる。
  1. bootmain.cをコンパイルする。nostdincオプションはシステムからヘッダーファイルを探さないようにするオプションで、Iオプションで指定されたディレクトリからヘッダーファイルを探す。つまりここではカレントディレクトリから探す。
  2. bootasm.Sをコンパイルする。
  3. bootblock.oとしてbootasm.oとbootmain.oをリンクする。Nオプションはtextセクションとdataセクションを読み書き可能にする。また、dataに関するセグメントをページに揃えないようにする。eオプションはエントリポイントを指定する。ここではstartをエントリポイントとしている。Ttextオプションはtextセグメントの開始アドレスを設定する。ここでは0x7C00にしている。0x7C00はBIOSがMBRをロードするアドレス。
  4. objdumpを使ってbootblock.oをディスアセンブリしてbootblock.asmを作成する。このファイルは使用しないが、アセンブリを見たいときに少し便利。
  5. objcopyを使ってbootblock.oからbootblockを作る。Sオプションは再配置情報とシンボル情報を削除する。Oオプションで出力をbinaryとする。jオプションは出力するセクションを指定する。つまりtextセクションだけをコピーする。
  6. sign.plを実行してbootblockをMBRにする。bootblockが510バイトより大きいときはエラーにする。そうでない場合は509バイト目までを0埋めして、510～511バイトにブートシグニチャ0x55AAを追加する。

Makefile
```Makefile
bootblock: bootasm.S bootmain.c
    $(CC) $(CFLAGS) -fno-pic -O -nostdinc -I. -c bootmain.c
    $(CC) $(CFLAGS) -fno-pic -nostdinc -I. -c bootasm.S
    $(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 -o bootblock.o bootasm.o bootmain.o                                                                   
    $(OBJDUMP) -S bootblock.o > bootblock.asm
    $(OBJCOPY) -S -O binary -j .text bootblock.o bootblock
    ./sign.pl bootblock
```
