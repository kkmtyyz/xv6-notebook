# 5.19. startothers関数
各APを起動し、GDT、ページング、IDT等の設定を行い、スケジューラを実行する。

APをスタートさせ、設定を済ませて一気にスケジューラの起動まで行う。
なのでmain関数の最後に呼ばれるmpmain関数もここで先に呼ばれる。
APで行う設定はほとんどBSPと同じように行う。

startothers関数を見る前に、APのエントリポイントとなるentryother.Sを見る。  
Makefileのターゲットentryotherを見ると、まずgccでentryother.Sからentryother.oを作成する。

各コマンドのオプションについては[「2. xv6.imgのビルド」](/chapter_02/02_00_xv6_img_build.md)で見た。  
ldのTtextオプションでTEXTセグメントの開始アドレスを0x7000とし、entryother.oからbootblockother.oを作成する。
objcopyでbootblockother.oからTEXTセクションのみをentryotherとしてコピーする。
出力にバイナリを指定しているため、entryotherに次の3つのシンボルが作成される。
- \_binary\_entryother\_start
- \_binary\_entryother\_end
- \_binary\_entryother\_size

最後にobjdumpでbootblockother.oを逆アセンブルし、entryother.asmを作成している。
カーネルには作成したバイナリのentryotherがリンクされる。

Makefile
```makefile
entryother: entryother.S
  $(CC) $(CFLAGS) -fno-pic -nostdinc -I. -c entryother.S
  $(LD) $(LDFLAGS) -N -e start -Ttext 0x7000 -o bootblockother.o entryother.o
  $(OBJCOPY) -S -O binary -j .text bootblockother.o entryother
  $(OBJDUMP) -S bootblockother.o > entryother.asm
```

[memmove関数](/chapter_05/05_09_consoleinit.md#memmove関数)を使用してentryother.Sのコードを物理アドレス0x7000にコピーする。
APではページングがまだ有効化されていないので、P2Vマクロを使用して仮想アドレスを求める必要がある。  

forループで[大域変数cpus](/chapter_05/05_04_mpinit.md)を走査し、APをひとつずつ起動する。
BSPの場合はcontinue。
このループはBSPで実行されているため、mycpu関数はBSPのcpu構造体を返す。  

APの起動時に使用するカーネルスタックとして変数stackに1ページ分のメモリを割り当てる。
割り当てには[kalloc関数](/chapter_05/05_03_kvmalloc.md#kalloc関数)を用いる。
大域変数kmemのuse\_lockフィールドは依然として0なので排他制御は行わない（kinit2関数で初めて1になる）。

entryotherを実行する際に渡す引数をスタックにセットする。
- **第一引数:** スタックの底のアドレス。
スタックのアドレスにカーネルスタックサイズ（4kB）を加算して求める。
- **第二引数:** main.cに定義されているmpenter関数のアドレス。関数ポインタとしてキャストして代入する。
- **第三引数:** main.cに定義されている[変数entrypgdir](/chapter_04/04_01_entry.md)のアドレス。
ラージページのページディレクトリで、0番と512番の2つのエントリが0ページ目（物理アドレス0から4MB分）を指している。

BSPはAPのcpu構造体のstartedフィールドが0でなくなるまでwhileループする。
startedフィールドが1になるまでの大まかな流れは次の通り。  
1. lapicstartap関数でAPを起動
2. codeとして渡したentryother.Sの実行
3. entryother.Sに第二引数として渡したmpenter関数の実行
4. mpmain関数でAPのcpu構造体のstartedフィールドに1を設定


main.c
```c
static void
startothers(void)
{
  extern uchar _binary_entryother_start[], _binary_entryother_size[];
  uchar *code;
  struct cpu *c;
  char *stack;

  // Write entry code to unused memory at 0x7000.
  // The linker has placed the image of entryother.S in
  // _binary_entryother_start.
  code = P2V(0x7000);
  memmove(code, _binary_entryother_start, (uint)_binary_entryother_size);

  for(c = cpus; c < cpus+ncpu; c++){
    if(c == mycpu())  // We've started already.
      continue;

    // Tell entryother.S what stack to use, where to enter, and what
    // pgdir to use. We cannot use kpgdir yet, because the AP processor
    // is running in low  memory, so we use entrypgdir for the APs too.
    stack = kalloc();
    *(void**)(code-4) = stack + KSTACKSIZE;
    *(void(**)(void))(code-8) = mpenter;
    *(int**)(code-12) = (void *) V2P(entrypgdir);

    lapicstartap(c->apicid, V2P(code));

    // wait for cpu to finish mpmain()
    while(c->started == 0)
      ;
  }
}

/* 略 */

__attribute__((__aligned__(PGSIZE)))
pde_t entrypgdir[NPDENTRIES] = {
  // Map VA's [0, 4MB) to PA's [0, 4MB)
  [0] = (0) | PTE_P | PTE_W | PTE_PS,
  // Map VA's [KERNBASE, KERNBASE+4MB) to PA's [0, 4MB)
  [KERNBASE>>PDXSHIFT] = (0) | PTE_P | PTE_W | PTE_PS,
};
```


## lapicstartap関数
この関数ではAPを起動し、entryother.Sを実行する。

APの起動方法は[「MultiProcessor Specification Version 1.4」（リンク14）](https://pdos.csail.mit.edu/6.828/2008/readings/ia32/MPspec.pdf)のB.4「Application Processor Startup」に記載されている。  
起動の流れは次のようになる。
1. BSPのBIOSシャットダウンコードを0x0Aに初期化し、warm reset vectorにAPリセット時に実行させるコードのアドレスを設定する。
BIOSシャットダウンコード（0x0A）は、リセット時にBIOSの初期化処理を行わず、EOI（End Of Interrupt割り込み終了の信号）なしで40:67（CS:IP）に格納されている4バイトのアドレスにジャンプする。
2. BSPから起動したいAPにINIT IPIを送る。
3. IPIの処理が終わるまで10ms待つ。
4. BSPから起動したいAPにSTARTUP IPIを送る。このとき、Vectorフィールドに実行開始アドレスを入れる。
5. IPIの処理が終わるまで200μs待つ。
6. 手順4と5をもう一度行う。INIT IPIとSTARTUP IPIは自動で再試行せず、OSはそれを正常に行う必要があるため2回呼び出す。
lapicstartap関数のコメントによると、2回目のSTARTUP IPIでのみAPを起動させるアーキテクチャも存在するらしい。

INIT IPIの使用方法は[lapicinit関数](/chapter_05/05_05_lapicinit.md#icrの設定)で見た。  
STARTUP IPIの使用方法は「MultiProcessor Specification Version 1.4」（リンク14）のB.4.2「USING STARTUP IPI」に記載されている。
このIPIは送信先のプロセッサをリアルモードで物理アドレス0x000VV000から実行する。
VVの部分は、ICRのVectorフィールドに設定された値が入る。

起動手順をlapicstartap関数に沿って読んでいく。
完全に上記手順に従っているわけではないため、行っていることがやや異なる。

**手順1:**  
BIOSのシャットダウンコードを0x0Aに初期化する。  
BIOSの設定を行うCMOSのポートは[「Bochs Developers Guide」（リンク9）](http://bochs.sourceforge.net/doc/docbook/development/index.html)によると、0x70～0x7Fであり0x70がCMOSのインデックスレジスタとなっている。
シャットダウンコードの設定はシャットダウンステータスバイト（0x0F）から行うことができ、0x0Aの場合は、リセット時に40:67（CS:IP）に格納されている4バイトのアドレスにジャンプする設定となる。  
リセット時にentryother.Sが実行されるよう、物理アドレス0x467にcode（引数addr）のアドレスを代入する。
リアルモードのセグメント機構ではセグメントレジスタが16bit、アドレスバスが20bitであるため、セグメントのアクセスではアドレスの下位4bitを0とする。
そのため0x7000（entryotherの開始アドレス）を4bit右シフトしている。
この辺りのことは[「初めて読む486」（書籍2）](/ref_books.md)に書いてある。

**手順2:**  
APにINIT IPIを2回送る。  
INIT IPIはlapicinit関数で使用したが、ここではLevelがAssertになっているため、もう一度見る。  
IPIについては[「Intel 64 and IA-32 architectures software developer's manual combined volumes: 1, 2A, 2B, 2C, 2D, 3A, 3B, 3C, 3D, and 4」（リンク8）](https://software.intel.com/content/www/us/en/develop/download/intel-64-and-ia-32-architectures-sdm-combined-volumes-1-2a-2b-2c-2d-3a-3b-3c-3d-and-4.html)のVol.3A「10.6.1 Interrupt Command Register (ICR)」に記載されている。  

1回目:  
LAPICのICR（Interrupt Command Register）（LAPICのインデックス0x310）に書き込みを行うことでIPIを送信する。
上半分（56bit目）にLAPIC IDを設定し、下半分にはINIT（0x500）、LEVEL（0x8000）、ASSERT（0x4000）を設定する（0b1100 0101 0000 0000）。  
8～10bitが0b101なので、Delivery ModeはINIT。  
11bitが0なのでDestination ModeはPhysical。  
14bitが1なのでLevelはAssert。  
15bitが1なのでTrigger ModeはLevel。  
Levelフラグが1（Assert）かつ、Delivery ModeのINITがLevel De-assertでないことから、INITリクエストを特定のプロセッサに送信することがわかる。
送信先はDestination ModeがPhysicalであることから、ICRの56～59bitで指定されたLAPICIDのプロセッサとなる。

INIT IPIの処理が終わるまで200μs待つ（[microdelay関数](/chapter_05/05_13_uartinit.md#uartputc関数)）。

2回目:  
Levelフラグを0（De-assert）でINIT IPIを送信する。
Delivery ModeがDe-assertなので送信先は全てのプロセッサとなる。

**手順3:**  
100μs待つ。  
「MultiProcessor Specification Version 1.4」（リンク14）には10msとあるが、lapicstartap関数のコメントにBochsでは遅すぎると記載がある。

**手順4:**  
STARTUP IPIを2回送る。  
ICRの56～59bitに送信先APのLAPIC IDを設定し、8～10bit目に0b110（Start-Up）を設定して、Vectorにentryother.Sのアドレス（0x7）を設定（実行を開始するアドレスVVの部分）する。

**手順5:**  
200μs待つ。  

関数内のAP起動手順は以上。
第一引数apicidで指定されたAPはSTARTUP IPIで起動され、entryother.Sが実行される。

lapic.c
```c
#define ICRLO   (0x0300/4)   // Interrupt Command
  #define INIT       0x00000500   // INIT/RESET
  #define STARTUP    0x00000600   // Startup IPI
  #define DELIVS     0x00001000   // Delivery status
  #define ASSERT     0x00004000   // Assert interrupt (vs deassert)
  #define DEASSERT   0x00000000
  #define LEVEL      0x00008000   // Level triggered
  #define BCAST      0x00080000   // Send to all APICs, including self.
  #define BUSY       0x00001000
  #define FIXED      0x00000000
#define ICRHI   (0x0310/4)   // Interrupt Command [63:32]

/* 略 */

#define CMOS_PORT    0x70
#define CMOS_RETURN  0x71

/* 略 */

void
lapicstartap(uchar apicid, uint addr)
{
  int i;
  ushort *wrv;

  // "The BSP must initialize CMOS shutdown code to 0AH
  // and the warm reset vector (DWORD based at 40:67) to point at
  // the AP startup code prior to the [universal startup algorithm]."
  outb(CMOS_PORT, 0xF);  // offset 0xF is shutdown code
  outb(CMOS_PORT+1, 0x0A);
  wrv = (ushort*)P2V((0x40<<4 | 0x67));  // Warm reset vector
  wrv[0] = 0;
  wrv[1] = addr >> 4;

  // "Universal startup algorithm."
  // Send INIT (level-triggered) interrupt to reset other CPU.
  lapicw(ICRHI, apicid<<24);
  lapicw(ICRLO, INIT | LEVEL | ASSERT);
  microdelay(200);
  lapicw(ICRLO, INIT | LEVEL);
  microdelay(100);    // should be 10ms, but too slow in Bochs!

  // Send startup IPI (twice!) to enter code.
  // Regular hardware is supposed to only accept a STARTUP
  // when it is in the halted state due to an INIT.  So the second
  // should be ignored, but it is part of the official Intel algorithm.
  // Bochs complains about the second one.  Too bad for Bochs.
  for(i = 0; i < 2; i++){
    lapicw(ICRHI, apicid<<24);
    lapicw(ICRLO, STARTUP | (addr>>12));
    microdelay(200);
  }
}
```

## entryother.S
この関数はSTARTUP IPIによりAPで実行される。  
概ね[bootasm.S](/chapter_03/03_01_bootasm.md)、[entry.S](/chapter_04/04_01_entry.md)と同様。  
GDTをロードし、プロテクトモードへ移行、ページングを有効化する。  
最後にスタックポインタをセットし、mpenter関数を呼び出す。

ラージページの設定まではbootasm.Sとentry.Sのコードと同じ。  
ページディレクトリの設定から見る。  
entryother.Sはobjcopyコマンドで作成されたバイナリとしてリンクされているため、main.cで定義されているentrypgdir変数が見えない。
そのため、startothers関数内であらかじめ第3引数の位置（start-12）にセットしておいたentrypgdirのアドレスを使用する。  
同様に、スタックポインタに設定するスタックのアドレスはstartothers関数にてkalloc関数で割り当てた1ページ分の領域を設定する。
これは第一引数の位置にセットしたので、 start-4 になる。  
最後にmpenter関数のアドレスは、startothers関数で第二引数の位置にセットしたので、start-8 になる。

entryother.S
```asm
.code16           
.globl start
start:
  cli            

  # Zero data segment registers DS, ES, and SS.
  xorw    %ax,%ax
  movw    %ax,%ds
  movw    %ax,%es
  movw    %ax,%ss

  # Switch from real to protected mode.  Use a bootstrap GDT that makes
  # virtual addresses map directly to physical addresses so that the
  # effective memory map doesn't change during the transition.
  lgdt    gdtdesc
  movl    %cr0, %eax
  orl     $CR0_PE, %eax
  movl    %eax, %cr0

  # Complete the transition to 32-bit protected mode by using a long jmp
  # to reload %cs and %eip.  The segment descriptors are set up with no
  # translation, so that the mapping is still the identity mapping.
  ljmpl    $(SEG_KCODE<<3), $(start32)

//PAGEBREAK!
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

  # Turn on page size extension for 4Mbyte pages
  movl    %cr4, %eax
  orl     $(CR4_PSE), %eax
  movl    %eax, %cr4
  # Use entrypgdir as our initial page table
  movl    (start-12), %eax
  movl    %eax, %cr3
  # Turn on paging.
  movl    %cr0, %eax
  orl     $(CR0_PE|CR0_PG|CR0_WP), %eax
  movl    %eax, %cr0

  # Switch to the stack allocated by startothers()
  movl    (start-4), %esp
  # Call mpenter()
  call	 *(start-8)

  movw    $0x8a00, %ax
  movw    %ax, %dx
  outw    %ax, %dx
  movw    $0x8ae0, %ax
  outw    %ax, %dx
spin:
  jmp     spin

.p2align 2
gdt:
  SEG_NULLASM
  SEG_ASM(STA_X|STA_R, 0, 0xffffffff)
  SEG_ASM(STA_W, 0, 0xffffffff)


gdtdesc:
  .word   (gdtdesc - gdt - 1)
  .long   gdt
```

## mpenter関数
BSPで行った設定を同様にAPにも行う。  
[switchkvm関数](/chapter_05/05_03_kvmalloc.md#switchkvm関数)でcr3にカーネル用のページディレクトリkpgdirのアドレスをセットする。
kpgdirはBSPと同じものが使用される。
[4kBページングとなる](/chapter_05/05_03_kvmalloc.md#walkpgdir関数)のも同様。  
[seginit関数](/chapter_05/05_06_seginit.md)でGDTの作成とロード、[lapicinit関数](/chapter_05/05_05_lapicinit.md)でLAPICの設定を行う。

main.c
```c
static void
mpenter(void)
{
  switchkvm();
  seginit();
  lapicinit();
  mpmain();
}
```

## mpmain関数
この関数はコンソールに「cpu1: starting 1」と表示し、IDTをロードしてcpu構造体のstartedフィールドの値を1にした後、スケジューラを呼び出す。
コンソールの文字列はLAPIC IDによって変わる。  
cpu構造体のstartedフィールドを[xchg関数](/chapter_05/05_10_lock.md#xchg関数)で1にする。
ここでxchg命令を使ってアトミックにstartedフィールドの値を更新する理由はわからない。  
この関数はBSPからmain関数の最後でも呼び出される。
scheduler関数はその時に見ることにする。

main.c
```c
static void
mpmain(void)
{
  cprintf("cpu%d: starting %d\n", cpuid(), cpuid());
  idtinit();       // load idt register
  xchg(&(mycpu()->started), 1); // tell startothers() we're up
  scheduler();     // start running processes
}
```

## cprintf関数
この関数は与えられたフォーマットでコンソールに文字を出力する。
フォーマットにはエスケープシーケンスを使用することが可能であり、第二引数以降の可変個の文字をフォーマットして挿入できる。

変数argpに可変長引数の先頭アドレスを代入する。
第一引数fmtのアドレスをuint分加算し、スタックの低い位置（高いアドレス）に有る第二引数を得る。  
fmtを1バイトずつ操作し、場合分けしながらコンソールに出力する。
- **%以外:** [consputc関数](/chapter_05/05_09_consoleinit.md#consolewrite関数)でコンソールに出力する。
- **0:** ループをbreak。
- **d:** printint関数で可変長引数から10進数符号ありでコンソールに出力する。
- **x, p:** printint関数で可変長引数から16進数符号なしでコンソールに出力する。
- **s:** 可変長引数から文字列を出力する。
1文字ずつ取り出し、値が0になるまでconsputc関数で1バイトずつコンソールに出力する。
1文字目が0の場合のみ文字列 "(null)" を出力する。
- **%:** consputc関数でコンソールに '%' を出力する。

console.c
```c
void
cprintf(char *fmt, ...)
{
  int i, c, locking;
  uint *argp;
  char *s;

  locking = cons.locking;
  if(locking)
    acquire(&cons.lock);

  if (fmt == 0)
    panic("null fmt");

  argp = (uint*)(void*)(&fmt + 1);
  for(i = 0; (c = fmt[i] & 0xff) != 0; i++){
    if(c != '%'){
      consputc(c);
      continue;
    }
    c = fmt[++i] & 0xff;
    if(c == 0)
      break;
    switch(c){
    case 'd':
      printint(*argp++, 10, 1);
      break;
    case 'x':
    case 'p':
      printint(*argp++, 16, 0);
      break;
    case 's':
      if((s = (char*)*argp++) == 0)
        s = "(null)";
      for(; *s; s++)
        consputc(*s);
      break;
    case '%':
      consputc('%');
      break;
    default:
      // Print unknown % sequence to draw attention.
      consputc('%');
      consputc(c);
      break;
    }
  }

  if(locking)
    release(&cons.lock);
}
```

## printint関数
この関数は整数xxを、基数baseとしてコンソールに出力する。
また、第三引数signが0以外の場合符号ありで出力する。

配列digitsを使用して整数xxを文字コードに変換し、配列bufに持つ。
bufは頭から詰めていき、お尻から出力する。

console.c
```c
static void
printint(int xx, int base, int sign)
{
  static char digits[] = "0123456789abcdef";
  char buf[16];
  int i;
  uint x;

  if(sign && (sign = xx < 0))
    x = -xx;
  else
    x = xx;

  i = 0;
  do{
    buf[i++] = digits[x % base];
  }while((x /= base) != 0);

  if(sign)
    buf[i++] = '-';

  while(--i >= 0)
    consputc(buf[i]);
}
```

## idtinit関数
この関数は[tvinit関数](/chapter_05/05_15_tvinit.md)で作成したIDTをlidt関数を通してlidt命令でロードする。

trap.c
```c
void
idtinit(void)
{
  lidt(idt, sizeof(idt));
}
```

x86.h
```asm
struct gatedesc;

static inline void
lidt(struct gatedesc *p, int size)
{
  volatile ushort pd[3];

  pd[0] = size-1;
  pd[1] = (uint)p;
  pd[2] = (uint)p >> 16;

  asm volatile("lidt (%0)" : : "r" (pd));
}
```
