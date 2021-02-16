# 3.1. bootasm.S
bootasm.SではGDTを作り、プロテクトモードへ切り替え、bootmain関数を呼び出す。

## A20ラインの有効化
最初にBIOSが立ち上がり、xv6.imgの最初の510～511バイト目の0x55AAを見てブートセクタだと判断して物理アドレス0x7C00にディスクから1セクタ（512バイト）分読み込む。
つまりxv6.imgのbootblockの部分が0x7C00に読み込まれる。  
bootblockの頭の方はbootasm.Sなので、startから実行が開始される。  
cli命令で割り込みを無効化する。  
axレジスタを使い、各セグメントレジスタ（ds, es, ss）の値を0に初期化する。  

次に、キーボードコントローラを使ってアドレスバスのA20ライン以上を使えるように設定を行う。  
このA20ラインや、セグメント機構、ページング、割り込み等については[「はじめて読む486」（書籍2）](docs/ref_books.md)に詳しく記載されている。
まだアドレスバスが0～19までしか無く、20bitで1MBのアドレスを使用していた時代、0xFFFFFの次0x100000へのアクセスで0x0にアクセスすることができたらしく、その性質を利用したプログラムも書かれていたらしい。その互換性を維持するために起動時はアドレスバスがA19までしか使用できないようになっているらしいので、この上限を開放する必要がある。
A20ラインの有効化にはいくつかの方法があり、その内のひとつとしてキーボードコントローラを使用した方法がある。

キーボードコントローラの仕様については[「Keyboard scancodes」（リンク7）](https://www.win.tue.nl/~aeb/linux/kbd/scancodes.html)の「The AT keyboard controller」に載っている。  
上記リンクによると、ポート0x64からの読み込みはステータスレジスタの内容となり、書き込みはコマンドとして解釈される。
また、ポート0x60への書き込みはコマンドのデータとして解釈される。

ラベルsata20.1では、ポート0x64からステータス0x2以外が読み出せるまでループしている。
ステータスは0bitがアウトプットバッファを示し、1bitがインプットバッファを示していて、0が空で1がフル。
バッファが空だった場合、ポート0x64にコマンド0xd1を書き込む。0xd1はアウトプットポートにデータを書き込むコマンド。
ラベルsata20.2のループでバッファが空であることを確認できるまで待ち、ポート0x60にデータ0xdfを書き込む。データ0xdfをアウトプットポートに書き込むと、A20ラインが有効になる。

bootasm.S
```asm
.code16                       # Assemble for 16-bit mode
.globl start
start:
  cli                         # BIOS enabled interrupts; disable

  # Zero data segment registers DS, ES, and SS.
  xorw    %ax,%ax             # Set %ax to zero
  movw    %ax,%ds             # -> Data Segment
  movw    %ax,%es             # -> Extra Segment
  movw    %ax,%ss             # -> Stack Segment

  # Physical address line A20 is tied to zero so that the first PCs 
  # with 2 MB would run software that assumed 1 MB.  Undo that.
seta20.1:
  inb     $0x64,%al               # Wait for not busy
  testb   $0x2,%al
  jnz     seta20.1

  movb    $0xd1,%al               # 0xd1 -> port 0x64
  outb    %al,$0x64

seta20.2:
  inb     $0x64,%al               # Wait for not busy
  testb   $0x2,%al
  jnz     seta20.2

  movb    $0xdf,%al               # 0xdf -> port 0x60
  outb    %al,$0x60
```

## GDTの作成とロード
プロテクトモードに切り替える準備をする。  
プロテクトモードのセグメント機構ではセグメントディスクリプタを使うので、事前にGDTを作ってlgdt命令で設定する必要がある。GDTに関しても[「はじめて読む486」（書籍2）](ref_books.md)に詳しく記載されている。  
lgdt命令ではgdtrにGDTのサイズとアドレスをセットする。
サイズは2バイトで、ラベル間の差を取って（gtddesc - gdt - 1）求める。
アドレスにはgdtラベルのアドレスをセットする。  
gdtラベルからGDTのエントリが書かれてる。全部で3エントリ。セグメントディスクリプタを作成するSEG\_ASMマクロはasm.hに定義されている。
p2align 2で2 * 2 = 4バイトでアラインメントする。  
セグメントディスクリプタの構造はプロセッサのマニュアル[「Intel 64 and IA-32 architectures software developer's manual combined volumes: 1, 2A, 2B, 2C, 2D, 3A, 3B, 3C, 3D, and 4」（リンク8）](https://software.intel.com/content/www/us/en/develop/download/intel-64-and-ia-32-architectures-sdm-combined-volumes-1-2a-2b-2c-2d-3a-3b-3c-3d-and-4.html)の「vol.3A 3.4.5 Segment Descriptors」に記載されてる。

作成されるセグメントディスクリプタの内容は、ビルドで作成されたbootblockバイナリを見て、マニュアルの図と照らし合わせるとわかりやすい。
SEG\_ASMマクロと引数の一部を電卓で計算すると、コードセグメントディスクリプタの方に値0x9Aが入っているのが分かるので、bootblockをxxdで開いてlessにパイプして9aでサーチすると位置0x60にGDTを見つけることができる。  
GDTに作成される3つのエントリは以下のようになっている。  
エントリ1: NULL。8バイト全て0。  
エントリ2: コードセグメントディスクリプタ。値は16進数で FF FF 00 00 00 9A CF 00。  
エントリ3: データセグメントディスクリプタ。値は16進数で FF FF 00 00 00 92 CF 00。  
コードセグメントディスクリプタの値はリミット値が0xFFFFF、セグメントベースが0x00000000、属性が0x9AC。属性は以下のようになってる。  
type 1100  
S    0  
DPL  01  
P    1  
AVL  1  
O    0  
D    0  
G    1  
リミット値は0xFFFFFだけど、Gフラグが1だからページ単位になって4K倍されるので0xFFFFF * 4096 = 4GB。
データセグメントディスクリプタの値もコードセグメントディスクリプタとだいたい同じになっている。属性は0x92C。

bootasm.S
```asm
# Bootstrap GDT
.p2align 2                                # force 4 byte alignment
gdt:
  SEG_NULLASM                             # null seg
  SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff)   # code seg
  SEG_ASM(STA_W, 0x0, 0xffffffff)         # data seg

gdtdesc:
  .word   (gdtdesc - gdt - 1)             # sizeof(gdt) - 1
  .long   gdt                             # address gdt
```

asm.h
```asm
#define SEG_NULLASM                                             \
        .word 0, 0;                                             \
        .byte 0, 0, 0, 0

// The 0xC0 means the limit is in 4096-byte units
// and (for executable segments) 32-bit mode.
#define SEG_ASM(type,base,lim)                                  \
        .word (((lim) >> 12) & 0xffff), ((base) & 0xffff);      \
        .byte (((base) >> 16) & 0xff), (0x90 | (type)),         \
                (0xC0 | (((lim) >> 28) & 0xf)), (((base) >> 24) & 0xff)

#define STA_X     0x8       // Executable segment
#define STA_W     0x2       // Writeable (non-executable segments)
#define STA_R     0x2       // Readable (executable segments)
```

## プロテクトモードへの切り替え
cr0のプロテクトモード有効フラグ（0bit）を立ててプロテクトモードに切り替える。
GDTのコードセグメントディスクリプタを使ってプロテクトモードでの実行を開始するために、ファージャンプをする。csレジスタの値はファージャンプでなければ変更できないため。
セグメントディスクリプタは1エントリ8バイトなので`ljmp $8, $start32`で、NULLの次のコードセグメントディスクリプタを指定する。

bootasm.S
```asm
  # Switch from real to protected mode.  Use a bootstrap GDT that makes
  # virtual addresses map directly to physical addresses so that the
  # effective memory map doesn't change during the transition.
  lgdt    gdtdesc
  movl    %cr0, %eax
  orl     $CR0_PE, %eax
  movl    %eax, %cr0

//PAGEBREAK!
  # Complete the transition to 32-bit protected mode by using a long jmp
  # to reload %cs and %eip.  The segment descriptors are set up with no
  # translation, so that the mapping is still the identity mapping.
  ljmp    $(SEG_KCODE<<3), $start32
```

mmu.h
```asm
#define SEG_KCODE 1  // kernel code
#define SEG_KDATA 2  // kernel data+stack
```

start32ラベルから32bit命令で実行が始まる。  
ds, es, ssにデータセグメントディスクリプタを設定する。  
fs, gsに0を設定する。  
espにstartのアドレスを設定する。startは0x7C00なのでそこから下に向かってスタックが伸びていくことになる。  
bootmain関数を呼び出す。bootmain関数はbootmain.cに定義されいる。  

bootasm.S
```asm
.code32  # Tell assembler to generate 32-bit code now.
start32:
  # Set up the protected-mode data segment registers
  movw    $(SEG_KDATA<<3), %ax    # Our data segment selector
  movw    %ax, %ds                # -> DS: Data Segment
  movw    %ax, %es                # -> ES: Extra Segment
  movw    %ax, %ss                # -> SS: Stack Segment
  movw    $0, %ax                 # Zero segments not ready for use
  movw    %ax, %fs                # -> FS
  movw    %ax, %gs                # -> GS

  # Set up the stack pointer and call into C.
  movl    $start, %esp
  call    bootmain
```

もしもbootmainがリターンした場合、ポート0x8a00にデータ0x8ae0を送る。  
Bochsのマニュアル[「Bochs Developers Guide」（リンク9）](http://bochs.sourceforge.net/doc/docbook/development/index.html)の「Chapter 3. Advanced debugger usage」から、ポート0x8A00はコマンドレジスタのサーバになっていて、コマンド0x8ae0はデバッガプロンプトに戻ることが分かる。そしてデバッガプロンプトに戻るということはCtrl-Cと同義であると書いてある。
デバッガプロンプトに戻った後、無限ループする。

bootasm.S
```asm
  # If bootmain returns (it shouldn't), trigger a Bochs
  # breakpoint if running under Bochs, then loop.
  movw    $0x8a00, %ax            # 0x8a00 -> port 0x8a00
  movw    %ax, %dx
  outw    %ax, %dx
  movw    $0x8ae0, %ax            # 0x8ae0 -> port 0x8a00
  outw    %ax, %dx
spin:
  jmp     spin
```
