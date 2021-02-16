# 3.2. bootmain関数
この関数はディスクからカーネル（elfバイナリ）を物理アドレス0x100000にロードし、entry関数を実行する。

[「2.1. ターゲットxv6.img」](/chapter_02/02_01_xv6_img.md)で見たように、kernelは1セクタ目から存在している。    
readseg関数で1セクタ目から4096バイト分をelfhdrポインタに読み込む。
ここでは読み込みを開始するオフセットに0を渡しているが、readseg関数は読み込みを1セクタ目から開始するので問題ない。  
マジックナンバーを確認し、読み込んだデータが少なくともelfヘッダを持っていることを確認する。  

elfの構造に関しては[「リンカ・ローダ 実践開発テクニック」（書籍1）](/ref_books.md)に詳しく記載されている。  
読み込んだデータ（カーネル）から、プログラムヘッダを取得する。elfヘッダのアドレス（0x10000）にelfhdr構造体のphoffフィールド（プログラムヘッダのオフセット）を加算することでプログラムヘッダが始まるアドレスが求められる。  
プログラムヘッダの個数はelfhdr構造体のphnumフィールドから得られる。  
プログラムヘッダの開始アドレスと個数が分かったので、for文で各プログラムヘッダをproghdr構造体に取り出し、以下のようにセグメントをディスクから読み込む。
  1. プログラムヘッダからロード先物理アドレス（paddrフィールド）を取得する。
  2. readseg関数でロード先物理アドレスに、ディスク上のセグメント開始位置（offフィールド）からセグメントサイズ（fileszフィールド）の分だけ読み込む。
  3. もしも、セグメントがメモリ上に展開されるサイズ（memszフィールド）が、セグメントサイズ（fileszフィールド）よりも大きい場合、stosb関数でセグメントの後ろをメモリ上に展開されるサイズまで0埋めする。

これでカーネルの全てのセグメントがメモリ上の適切な位置にロードされた。  
elfバイナリ（カーネル）のエントリーポイントをelfヘッダのentryフィールドから取得する。関数として呼び出すため、関数ポインタにキャストして取得している。ここではエントリーポイントはentry.Sの\_startラベル。  

最後にエントリーポイントを関数として呼び出し、カーネルの実行を開始する。

elf.h
```c
#define ELF_MAGIC 0x464C457FU  // "\x7FELF" in little endian

// File header
struct elfhdr {
  uint magic;  // must equal ELF_MAGIC
  uchar elf[12];
  ushort type;
  ushort machine;
  uint version;
  uint entry;
  uint phoff;
  uint shoff;
  uint flags;
  ushort ehsize;
  ushort phentsize;
  ushort phnum;
  ushort shentsize;
  ushort shnum;
  ushort shstrndx;
};

// Program section header
struct proghdr {
  uint type;
  uint off;
  uint vaddr;
  uint paddr;
  uint filesz;
  uint memsz;
  uint flags;
  uint align;
};
```

bootmain.c
```c
void
bootmain(void)
{
  struct elfhdr *elf;
  struct proghdr *ph, *eph;
  void (*entry)(void);
  uchar* pa;

  elf = (struct elfhdr*)0x10000;  // scratch space

  // Read 1st page off disk
  readseg((uchar*)elf, 4096, 0);

  // Is this an ELF executable?
  if(elf->magic != ELF_MAGIC)
    return;  // let bootasm.S handle error

  // Load each program segment (ignores ph flags).
  ph = (struct proghdr*)((uchar*)elf + elf->phoff);
  eph = ph + elf->phnum;
  for(; ph < eph; ph++){
    pa = (uchar*)ph->paddr;
    readseg(pa, ph->filesz, ph->off);
    if(ph->memsz > ph->filesz)
      stosb(pa + ph->filesz, 0, ph->memsz - ph->filesz);
  }

  // Call the entry point from the ELF header.
  // Does not return!
  entry = (void(*)(void))(elf->entry);
  entry();
}
```

プログラムヘッダとセクションヘッダの情報はobjdumpで確認できるので、以下のようにここで処理されているプログラムヘッダの値を確認できる。  
プログラムヘッダは次の3エントリ。仮想アドレスと物理アドレスは[「2.4. ターゲットkernel」](/chapter_02/02_04_kernel.md)で見たリンカスクリプトkernel.ldにより設定されている。  
  1. textセクション。物理アドレス0x100000にロードする。  
  2. dataセクション。物理アドレス0x108000にロードする。  
  3. スタック。サイズが0。  

`objdump -ph kernel | less`の出力結果
```
kernel:     file format elf32-i386

Program Header:
    LOAD off    0x00001000 vaddr 0x80100000 paddr 0x00100000 align 2**12
         filesz 0x0000788c memsz 0x0000788c flags r-x
    LOAD off    0x00009000 vaddr 0x80108000 paddr 0x00108000 align 2**12
         filesz 0x00002516 memsz 0x0000d4a8 flags rw-
   STACK off    0x00000000 vaddr 0x00000000 paddr 0x00000000 align 2**4
         filesz 0x00000000 memsz 0x00000000 flags rwx

Sections:
Idx Name          Size      VMA       LMA       File off  Algn
  0 .text         00006e92  80100000  00100000  00001000  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  1 .rodata       000009ec  80106ea0  00106ea0  00007ea0  2**5
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  2 .data         00002516  80108000  00108000  00009000  2**12
                  CONTENTS, ALLOC, LOAD, DATA
  3 .bss          0000af88  8010a520  0010a520  0000b516  2**5
                  ALLOC
  4 .debug_line   00002694  00000000  00000000  0000b516  2**0
                  CONTENTS, READONLY, DEBUGGING
  5 .debug_info   000104e2  00000000  00000000  0000dbaa  2**0
                  CONTENTS, READONLY, DEBUGGING
  6 .debug_abbrev 0000390e  00000000  00000000  0001e08c  2**0
                  CONTENTS, READONLY, DEBUGGING
  7 .debug_aranges 000003a8  00000000  00000000  000219a0  2**3
                  CONTENTS, READONLY, DEBUGGING
  8 .debug_loc    00005239  00000000  00000000  00021d48  2**0
                  CONTENTS, READONLY, DEBUGGING
  9 .debug_ranges 00000748  00000000  00000000  00026f81  2**0
                  CONTENTS, READONLY, DEBUGGING
 10 .debug_str    00000e48  00000000  00000000  000276c9  2**0
                  CONTENTS, READONLY, DEBUGGING
 11 .comment      0000002d  00000000  00000000  00028511  2**0
                  CONTENTS, READONLY
```

また、elfヘッダのentryフィールドが示すエントリーポイントのアドレスはreadelfで確認できる。
textセクション開始位置の少し後ろ0x10000cにエントリーポイントがある。

`readelf -h kernel`
```
ELF Header:
  Magic:   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF32
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Intel 80386
  Version:                           0x1
  Entry point address:               0x10000c
  Start of program headers:          52 (bytes into file)
  Start of section headers:          149308 (bytes into file)
  Flags:                             0x0
  Size of this header:               52 (bytes)
  Size of program headers:           32 (bytes)
  Number of program headers:         4
  Size of section headers:           40 (bytes)
  Number of section headers:         16
  Section header string table index: 15
```


## readseg関数
この関数はサイズを指定してディスクからデータを読み込む。

データの読み込み先物理アドレス（pa）と、読み込むバイト数（count）、読み込みを開始するディスクのオフセットバイト数（offset）を引数として受けとる。  
ディスクからの読み込みは1セクタ（512バイト）ずつだが、読み込みたいデータがセクタの途中から開始される場合がある。offsetが512で割り切れない場合がそう。
その場合のために、読み込み先アドレスをセクタ内の読み込み開始位置までの分だけ（offset % 512）下げておく。これで引数で与えられた読み込み先アドレスに丁度offsetバイト目のデータが入る。  
また、読み込みはセクタ番号を指定して行い、offsetをその番号の指定に使用するため、セクタサイズで割っておく。このとき、1加算することでブートセクタを含まないようにしている。
つまりoffsetの値が0でも、データの読み込みは1セクタ目から開始される。  
データの読み込みにはreadsect関数を用いる。  

bootmain.c
```c
#define SECTSIZE  512

/* 略 */

void
readseg(uchar* pa, uint count, uint offset)
{
  uchar* epa;

  epa = pa + count;

  // Round down to sector boundary.
  pa -= offset % SECTSIZE;

  // Translate from bytes to sectors; kernel starts at sector 1.
  offset = (offset / SECTSIZE) + 1;

  // If this is too slow, we could read lots of sectors at a time.
  // We'd write more to memory than asked, but it doesn't matter --
  // we load in increasing order.
  for(; pa < epa; pa += SECTSIZE, offset++)
    readsect(pa, offset);
}
```

## readsect関数
この関数はセクタを指定してディスクからデータを読み込む。

IOポートの読み書きにはinb関数やoutb関数を使用する。
ポート番号の意味は[「OSDev I/O Ports」（リンク10）](https://wiki.osdev.org/I/O_Ports)にリストされている。
また、個々のポートの働きに関しては[「XT, AT and PS/2	 I/O port addresses」（リンク11）](http://bochs.sourceforge.net/techspec/PORTS.LST)に詳しく載っている。  
ディスクコントローラの読み書きを行う前に、waitdisk関数を使用してディスクの準備ができるまで待機する。
waitdisk関数はwhileループを使い、ディスクコントローラのステータスレジスタ（0x1F7）の6, 7bitが1, 0でなくなるまで待つ。6bitはディスクのreadyを示し、7bitはコントローラがコマンドを実行中かどうかを示している。つまり状態がreadyかつ、コマンド実行中でなくなるまで待機する。  
ポート0x1F7は読み込み時にはステータスレジスタ、書き込み時にはコマンドレジスタへのアクセスとなる。

readsect関数は読み込み先アドレス（dst）とセクタ番号（offset）を引数として受け取る。  
ポート0x1F2から0x1F7に書き込みを行い、以下の内容のコマンドを発行する。  
0x1F2: 1セクタ分  
0x1F3: offset番目のセクタ  
0x1F4, 0x1F5: offsetの3バイト目 + offsetの2バイト目で表されるシリンダー  
0x1F6: offsetの4バイト目のビットで示されるヘッド  
0x1F7: セクタをリードする（再試行ありで）  
コマンド実行終了までwaitdisk関数で待機し、insl関数でデータレジスタ（0x1f0）からdstに512 / 4 = 128 * 4バイト = 512バイト読み込む。  

bootmain.c
```c
void
waitdisk(void)
{
  // Wait for disk ready.
  while((inb(0x1F7) & 0xC0) != 0x40)
    ;
}

// Read a single sector at offset into dst.
void
readsect(void *dst, uint offset)
{
  // Issue command.
  waitdisk();
  outb(0x1F2, 1);   // count = 1
  outb(0x1F3, offset);
  outb(0x1F4, offset >> 8);
  outb(0x1F5, offset >> 16);
  outb(0x1F6, (offset >> 24) | 0xE0);
  outb(0x1F7, 0x20);  // cmd 0x20 - read sectors

  // Read data.
  waitdisk();
  insl(0x1F0, dst, SECTSIZE/4);
}
```

## inb関数、outb関数、insl関数
inb関数はポートから1バイト読み込む。  
outb関数はポートに1バイト書き込む。  
insl関数はポートから指定回数分だけ4バイトずつ読み込む。  

これらの関数ではインラインアセンブリを使用する。GCCのインラインアセンブリについては[「Using the GNU Compiler Collection (GCC)」（リンク3）](https://gcc.gnu.org/onlinedocs/gcc-6.5.0/gcc/)の[「6.44 How to Use Inline Assembly Language in C Code」](https://gcc.gnu.org/onlinedocs/gcc-6.5.0/gcc/Using-Assembly-Language-with-C.html#Using-Assembly-Language-with-C)で説明されている。  
また、x86の命令については[「Intel 64 and IA-32 architectures software developer's manual combined volumes: 1, 2A, 2B, 2C, 2D, 3A, 3B, 3C, 3D, and 4」（リンク8）](https://software.intel.com/content/www/us/en/develop/download/intel-64-and-ia-32-architectures-sdm-combined-volumes-1-2a-2b-2c-2d-3a-3b-3c-3d-and-4.html)の「Vol.2 Instruction Set Reference, A-Z」で確認できる。  

x86.h
```c
static inline uchar
inb(ushort port)
{
  uchar data;

  asm volatile("in %1,%0" : "=a" (data) : "d" (port));
  return data;
}

static inline void
insl(int port, void *addr, int cnt)
{
  asm volatile("cld; rep insl" :
               "=D" (addr), "=c" (cnt) :
               "d" (port), "0" (addr), "1" (cnt) :
               "memory", "cc");
}

static inline void
outb(ushort port, uchar data)
{
  asm volatile("out %0,%1" : : "a" (data), "d" (port));
}
```

insl関数の動きを見ると、以下のようになっている。  
cld命令でEFLAGSのDFフラグを0にすることで、文字列操作が行われるとesiとediがインクリメントされるようになる。このため出力先edi(addr)が毎回4バイトずつずれてくれる。  
拡張インラインアセンブリの入力にある数字（ここでは0と1）は、番号と一致するオペランドを使うことを示している。
つまりここでは出力が「”=D(addr), “=c”(cnt)」なので、入力は「”d”(port), “D”(addr), “c”(cnt)」と書き直せる。ediとecx
が入力としても出力としても使用される。  
insl命令はedxの示すポートから4バイトをediに読み込む。
また、命令にrepがつくと、文字列操作の実行毎にカウントレジスタ（ecx）がデクリメントされ、指定された回数だけ命令をリピートするようになる。
ここで、一度出力と入力を整理する。  
出力: edi(addr), ecx(cnt)  
入力: edx(port), edi(addr), ecx(cnt)  
insl命令では出力: edi(addr)と入力: edx(port)を使う。rep命令はecxを使う。  
一見入力のedi(addr)とecx(cnt)が不要に見える。ディスアセンブリされたbootblock.asmを見ると、この部分は次のようになっていて、入力としてのediとecxはますます不要に思える。  

bootblock.asm
```asm
  asm volatile("cld; rep insl" :
    7ce4:	8b 7d 08             	mov    0x8(%ebp),%edi
    7ce7:	b9 80 00 00 00       	mov    $0x80,%ecx
    7cec:	ba f0 01 00 00       	mov    $0x1f0,%edx
    7cf1:	fc                   	cld    
    7cf2:	f3 6d                	rep insl (%dx),%es:(%edi)
  insl(0x1F0, dst, SECTSIZE/4);
```

しかし、[「ProgrammerSought Gnu embedded assembly, inline assembly detailed introduction」（リンク12）](https://programmersought.com/article/74671233226/)によると、repでループする過程において、ループ毎のedi(addr)とecx(cnt)が同一のものであるということをコンパイラに伝えるために必要らしい。  
このrepを使用したパターンは今後も出てくる。


## stosb関数
この関数は値（data）を指定バイト分（cnt）だけアドレス（addr）に書き込む。  

repを使用したパターンなので、insl関数と同様の動きになる。

x86.h
```c
> 42 static inline void
> 43 stosb(void *addr, int data, int cnt)
> 44 {
> 45   asm volatile("cld; rep stosb" :
> 46                "=D" (addr), "=c" (cnt) :
> 47                "0" (addr), "1" (cnt), "a" (data) :
> 48                "memory", "cc");
> 49 }
```
これで物理アドレス0x100000にカーネルをロードすることができた。

