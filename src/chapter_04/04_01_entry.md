# 4.1. entry.S
entry.Sではページングを有効化し、ページディレクトリを作成して、main関数を呼び出す。

エントリーポイントは[「2.4. ターゲットkernel」](https://kkmtyyz.github.io/xv6-notebook/chapter_02/02_04_kernel.html)で見たリンカスクリプトkernel.ldにより\_startに設定されている。\_startのアドレスは0x10000cだったことを[3.2. bootmain関数](https://kkmtyyz.github.io/xv6-notebook/chapter_03/03_02_bootmain.html)で確認した。  
今、カーネルを実行していくためにentry.Sのentryラベルから実行を開始したい。
しかし、リンカスクリプトによりカーネルは仮想アドレス0x80100000で実行されるようにリンクされている。
これは[「xv6 a simple, Unix-like teaching operating system」](https://pdos.csail.mit.edu/6.828/2018/xv6/book-rev11.pdf)の「Figure 1-3. Layout of a Virtual address space」のように、仮想アドレスの上の方にカーネルを置きたいからである。
このためシンボルのアドレスを見ると、entryは0x8010000cで実行されるようになっている。
しかし、kernelのtextのロード先は物理アドレス0x100000なので、実際にはentryは0x100000の近くに配置されている。
なので、`V2P_WO(entry)`でentryの実際のアドレスを求め、\_startのアドレス0x10000cに代入することでentryを実行する。  
以降同様の目的でV2P\_WOマクロを度々使用する。

`readelf -s kernel | grep -e start -e entry`
```
    46: 801026e6   402 FUNC    LOCAL  DEFAULT    1 idestart
    75: 8010393f   196 FUNC    LOCAL  DEFAULT    1 startothers
   131: 8010000c     0 NOTYPE  GLOBAL DEFAULT    1 entry
   348: 8010a000  4096 OBJECT  GLOBAL DEFAULT    4 entrypgdir
   349: 0010000c     0 NOTYPE  GLOBAL DEFAULT    1 _start
   391: 0000008a     0 NOTYPE  GLOBAL DEFAULT  ABS _binary_entryother_size
   429: 8010b4ec     0 NOTYPE  GLOBAL DEFAULT    4 _binary_entryother_start
   492: 8010b4c0     0 NOTYPE  GLOBAL DEFAULT    4 _binary_initcode_start
   504: 8010b576     0 NOTYPE  GLOBAL DEFAULT    4 _binary_entryother_end
   560: 80103022   227 FUNC    GLOBAL DEFAULT    1 lapicstartap
```
memlayout.h
```c
#define KERNBASE 0x80000000         // First kernel virtual address

/* 略 */

#define V2P_WO(x) ((x) - KERNBASE)    // same as V2P, but without casts
```

entry.S
```asm
.globl _start
_start = V2P_WO(entry)

# Entering xv6 on boot processor, with paging off.
.globl entry
entry:
```

ページング機構を有効にする。  
なお、ページング機構を有効にしてもセグメント機構は無効にならない。
セグメント機構は無効にできないので常に有効になっている。
ページサイズは通常4kBだが、cr4の4bitを1にすると4MBにできる。ここでは4MBページングを有効にする。
[「Intel 64 and IA-32 architectures software developer's manual combined volumes: 1, 2A, 2B, 2C, 2D, 3A, 3B, 3C, 3D, and 4」](https://software.intel.com/content/www/us/en/develop/download/intel-64-and-ia-32-architectures-sdm-combined-volumes-1-2a-2b-2c-2d-3a-3b-3c-3d-and-4.html)の「Vol.3A 4.3 32-BIT PAGING」にページングに関して記載されているが、cr4.PSEが何bitなのかが分からなかった。
しかし、[「Wikipedia Control register」（リンク13）](https://en.wikipedia.org/wiki/Control_register)によると4bitがPSEらしく、実際コードもそうなっている。

ページディレクトリには、main.cに定義されているentrypgdirを使う。  
ページング機構ではページディレクトリのアドレスをcr3に設定することになっているため、entrypgdirの物理アドレスを設定する。
ページディレクトリentrypgdirは、0番と512番の2つのエントリを持つ。
各エントリがページを指しており、ここでは0番目も512番目も同じ0ページ目（物理アドレス0から4MB分）を指している。
また、ページがメモリ上に存在し（PTE\_P）、書き込み可能で（PTE\_W）、グローバルなページである（PTE\_PS）ことを示している。
これらページディレクトリエントリの各部位の意味や、仮想アドレスの変換方法についてはSDMの「Vol.3A 4.3 32-Bit Paging」に図付きで記述されている。
4MBページでの仮想アドレスの変換方法はSMDの「Figure 4-3. Linear-Address Translation to a 4-MByte Page using 32-bit Paging」の通り。
仮想アドレスの上位10bitがページディレクトリのエントリ番号を示しており、今回はページディレクトリエントリのフラグ部分以外が0なので、ページフレーム0の物理アドレス0x0からページが開始される。

512エントリ目はこの後main関数以降の実行アドレスが高い（0x8000000等）ため必要であり。
0エントリ目はページングを有効にした瞬間からページング機構を用いたアドレス変換が始まるので、低い位置で動いている今（0x100000等）必要となる。

cr0の31bitを1にしてページング機構を有効にし、16bitも1にして書き込み可能ページへの書き込みを特権レベル0でも禁止する。

main関数を実行する前に、今後使用するスタックを準備する。  
.commでスタック分の領域が作られ、KSTACKSIZEが4096なのでスタックサイズは4kB。
スタックは上から下に伸びるので、espにはstack + 4096したアドレスを設定する。
.commなのでbssセクションに作成される。リンカスクリプトではbssセクションは後ろの方に作ったので、カーネルの後ろ、データのさらに後ろをスタックとして使用することになる。


memlayout.h
```c
#define KERNBASE 0x80000000         // First kernel virtual address

/* 略 */

#define V2P_WO(x) ((x) - KERNBASE)    // same as V2P, but without casts
```

param.h
```c
#define KSTACKSIZE 4096  // size of per-process kernel stack
```

entry.S
```asm
.globl _start
_start = V2P_WO(entry)

# Entering xv6 on boot processor, with paging off.
.globl entry
entry:
  # Turn on page size extension for 4Mbyte pages
  movl    %cr4, %eax
  orl     $(CR4_PSE), %eax
  movl    %eax, %cr4
  # Set page directory
  movl    $(V2P_WO(entrypgdir)), %eax
  movl    %eax, %cr3
  # Turn on paging.
  movl    %cr0, %eax
  orl     $(CR0_PG|CR0_WP), %eax
  movl    %eax, %cr0

  # Set up the stack pointer.
  movl $(stack + KSTACKSIZE), %esp

  # Jump to main(), and switch to executing at
  # high addresses. The indirect call is needed because
  # the assembler produces a PC-relative instruction
  # for a direct jump.
  mov $main, %eax
  jmp *%eax

.comm stack, KSTACKSIZE
```

main.c
```c
__attribute__((__aligned__(PGSIZE)))
pde_t entrypgdir[NPDENTRIES] = {4MB
  // Map VA's [0, 4MB) to PA's [0, 4MB)
  [0] = (0) | PTE_P | PTE_W | PTE_PS,
  // Map VA's [KERNBASE, KERNBASE+4MB) to PA's [0, 4MB)
  [KERNBASE>>PDXSHIFT] = (0) | PTE_P | PTE_W | PTE_PS,
};
```

mmu.h
```c
#define NPDENTRIES      1024    // # directory entries per page directory
#define PGSIZE          4096    // bytes mapped by a page
#define PDXSHIFT        22      // offset of PDX in a linear address
// Page table/directory entry flags.
#define PTE_P           0x001   // Present
#define PTE_W           0x002   // Writeable
#define PTE_U           0x004   // User
#define PTE_PS          0x080   // Page Size
```

entry.Sの最後、main関数にジャンプする。  
main関数が実際にロードされているアドレスはELFのテキストセグメントを0x100000に読み込んだので、おそらくそのちょっと先くらいだろう。
仮想アドレスはreadelfで確認すると0x80103853になっている。  

`readelf -s kernel | grep main`
```
    73: 00000000     0 FILE    LOCAL  DEFAULT  ABS main.c
    76: 801038f8    71 FUNC    LOCAL  DEFAULT    1 mpmain
   415: 80103853   139 FUNC    GLOBAL DEFAULT    1 main
```

ページングが有効になっているので、先ほどのページディレクトリを使って以下のようにmainのアドレスが変換される。  
  1. 仮想アドレス0x80103853(main)から上位10bitを取り出す。0b1000 0000 00になる。
10進数で512なので、ページディレクトリの512番目のエントリを使用する。
  2. 512番目のエントリはフラグ以外0なので、ページフレームを示すbitも全て0となり、0番目のページフレームを使うことが分かる。
  3. 仮想アドレス0x80103853(main)の下位22bitを取り出す。0b01 0000 0011 1000 0101 0011になる。
これは0x103853なので、0ページ目の0x103853に変換される。つまりmainの物理アドレスは0x103853となる。

この変換はSDMの「Figure 4-3. Linear-Address Translation to a 4-MByte Page using 32-bit Paging」を見ながら行った。

これでページングを有効にし、main関数にジャンプすることができた。
