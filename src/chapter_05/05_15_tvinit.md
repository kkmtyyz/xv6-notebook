# 5.15. tvinit関数
大域変数idtに256個のゲートディスクリプタを作成する。

IDT（Interrupt Descriptor Table）を作成する。  
IDTについては[「Intel 64 and IA-32 architectures software developer's manual combined volumes: 1, 2A, 2B, 2C, 2D, 3A, 3B, 3C, 3D, and 4」（リンク8）](https://software.intel.com/content/www/us/en/develop/download/intel-64-and-ia-32-architectures-sdm-combined-volumes-1-2a-2b-2c-2d-3a-3b-3c-3d-and-4.html)のVol.3 Chapter6.10「Interrupt Descriptor Table (IDT)」で説明されている。
ここでは、IDTはgatedesc構造体の配列になっており、大域変数idtで宣言されている。
Intel SDMによると256個の割り込みベクタを持てるため、idtの要素数も256。

IDTに格納される各ゲートディスクリプタの構造はIntel SDMの図6-2「IDT Gate Descriptors」に記載されており、ここで使用するのはInterrupt GateとTrap Gate。
割り込みにはInterrupt Gateを使用し、システムコールにはTrap Gateを使用する。

gatedesc構造体はmmu.hで定義されている。  
- **off_15_0:** 割り込みベクタのアドレスの下半分を設定する。  
- **cs:** GDT上のエントリのインデックスを設定する。
セグメントディスクリプタは8バイトなので8の倍数。  
- **args:** 予約済みなので0埋め。  
- **rsv1:** 全て0。  
- **type:** Interrupt Gateの場合は0b1110（0xE）、Trap Gateの場合は0b1111（0xF）を設定する。
Intel SDMの図で確認すると3bitがDになり、ゲートディスクリプタのサイズが32bitの場合1、16bitの場合0になる。
なのでここでは1で固定。  
- **s:** 12bitに当たるので0固定。  
- **dpl:** ディスクリプタ特権レベル（Descriptor Privilege Level）なので、Interrupt Gateの場合0、Trap Gateの場合3を設定する。
特権レベルについてはVol.3 Chapter5.5「PRIVILEGE LEVELS」で説明されており、カーネルが0、ユーザが3。  
- **p:** セグメント存在フラグ。今回はセグメントがメモリ上に存在するので1を設定する。  
- **off_31_16:** 割り込みベクタのアドレスの上半分を設定する。

SETGATEマクロで各ゲートディスクリプタを作成する。

trap.c
```c
struct gatedesc idt[256];
```

mmu.h
```c
struct gatedesc {
  uint off_15_0 : 16;   // low 16 bits of offset in segment
  uint cs : 16;         // code segment selector
  uint args : 5;        // # args, 0 for interrupt/trap gates
  uint rsv1 : 3;        // reserved(should be zero I guess)
  uint type : 4;        // type(STS_{IG32,TG32})
  uint s : 1;           // must be 0 (system)
  uint dpl : 2;         // descriptor(meaning new) privilege level
  uint p : 1;           // Present
  uint off_31_16 : 16;  // high bits of offset in segment
};

/* 略 */

#define SETGATE(gate, istrap, sel, off, d)                \
{                                                         \
  (gate).off_15_0 = (uint)(off) & 0xffff;                \
  (gate).cs = (sel);                                      \
  (gate).args = 0;                                        \
  (gate).rsv1 = 0;                                        \
  (gate).type = (istrap) ? STS_TG32 : STS_IG32;           \
  (gate).s = 0;                                           \
  (gate).dpl = (d);                                       \
  (gate).p = 1;                                           \
  (gate).off_31_16 = (uint)(off) >> 16;                  \
}
```

割り込みベクタテーブルはvectors.Sに定義されている256個の関数のアドレスが、同ファイル内のvectorsラベル以下に列挙されている。
関数はvector0からvector255まであり、スタックに0とIRQ番号をプッシュし、alltraps関数に跳んでいる。
alltraps関数はtrapasm.Sで定義されている。
vectors.Sは[make時](/chapter_02/02_04_kernel.md)にvectors.plを実行することで作られる。  
trap.cで、extern宣言として符号なし整数型の配列vectorsで割り込みベクタテーブルを参照している。

vectors.S
```asm
.globl alltraps
.globl vector0
vector0:
  pushl $0
  pushl $0
  jmp alltraps
.globl vector1

# 略

.globl vector255
vector255:
  pushl $0
  pushl $255
  jmp alltraps

# vector table
.data
.globl vectors
vectors:
  .long vector0

# 略

  .long vector255
```

trap.c
```c
extern uint vectors[];  // in vectors.S: array of 256 entry pointers
```

tvinit関数では、まずforループで配列idtに割り込みベクタテーブルの全エントリ分（256個）の割り込みゲートディスクリプタを作成する。
DPLは0。  
次に、システムコールにはIRQ64番を使うので、idtの64番目を上書きする形でトラップゲートディスクリプタをDPL3で作成する。  
最後にtickslockのロックを初期化する。

trap.c
```c
void
tvinit(void)
{
  int i;

  for(i = 0; i < 256; i++)
    SETGATE(idt[i], 0, SEG_KCODE<<3, vectors[i], 0);
  SETGATE(idt[T_SYSCALL], 1, SEG_KCODE<<3, vectors[T_SYSCALL], DPL_USER);

  initlock(&tickslock, "time");
}
```


## tick
tickは一般的に一定周期で時間を刻むことを言い、ここでも同様に使われる。

符号なしint型の大域変数ticksと、そのロックに使用するtickslockが定義されている。
使用箇所をgrepすると、タイマー割り込みに関する部分とシステムコールsys\_sleep内でのみ使用されており、インクリメントだけで代入は行われない。  
sysproc.cに定義されているsys\_sleep関数でその使われ方を見ることができる。
この関数はticksが引数n分だけインクリメントされるまで、現在のプロセスをスリープ状態にする。  
はじめに、argint関数を使用して自身に与えられたint型の引数を変数nに取り出す。  
そして現在のticksの値をローカル変数ticks0に保存しておく。  
whileループで `ticks - ticks0 < n` が偽になるまでプロセスをスリープする。
起床するためのチャネルにはticksのアドレスを用いる。  
ticksはcpu0からのタイマー割り込みによりインクリメントされるので、その度に起床されることになる。

trap.c
```c
struct spinlock tickslock;
uint ticks;
```

sysproc.c
```c
int
sys_sleep(void)
{
  int n;
  uint ticks0;

  if(argint(0, &n) < 0)
    return -1;
  acquire(&tickslock);
  ticks0 = ticks;
  while(ticks - ticks0 < n){
    if(myproc()->killed){
      release(&tickslock);
      return -1;
    }
    sleep(&ticks, &tickslock);
  }
  release(&tickslock);
  return 0;
}
```

ticksがインクリメントされる部分だけ合わせて見ることにする。  
割り込みやトラップが生じるとIDTや割り込みベクタテーブル等を経由してtrapasm.Sのalltraps関数からtrap.cのtrap関数が呼び出されれる。  
trap関数はIRQ番号によって分岐する。  
ticksはタイマー割り込みかつcpu0からの割り込みである場合にのみインクリメントされる。  
その後ticksのアドレスをチャネルとしてスリープ状態のプロセスを起床させる。  
LAPICタイマの設定は[lapicinit](/chapter_05/05_05_lapicinit.md#LAPICタイマの設定)で見た。  

trap.c
```c
  switch(tf->trapno){
  case T_IRQ0 + IRQ_TIMER:
    if(cpuid() == 0){
      acquire(&tickslock);
      ticks++;
      wakeup(&ticks);
      release(&tickslock);
    }
    lapiceoi();
    break;
```
