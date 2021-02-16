# 5.5. lapicinit関数
この関数はLAPICを有効にし、スプリアス割り込みを無効化。APICタイマが10000000からバスクロックが進むごとにカウントダウンされ、0になるとIRQ32で割り込みをかけ、再度カウントダウンを始めるよう設定。LINT0とLINT1ピン、パフォーマンスモニタリングカウンタも無効化。割り込みエラーの際にIRQ51で割り込みを発生するように設定。ESR、EOIレジスタをリセット。ICRでAPにINIT IPIを送信し、Arb IDをLAPIC IDと同じ値に設定。TPRに0を設定する。

LAPICの設定を行うので、[「Intel 64 and IA-32 architectures software developer's manual combined volumes: 1, 2A, 2B, 2C, 2D, 3A, 3B, 3C, 3D, and 4」（リンク8）](https://software.intel.com/content/www/us/en/develop/download/intel-64-and-ia-32-architectures-sdm-combined-volumes-1-2a-2b-2c-2d-3a-3b-3c-3d-and-4.html)の「Vol.3 CHAPTER 10 ADVANCED PROGRAMMABLE INTERRUPT CONTROLLER (APIC)」を参照する。  
MPの場合、外部からの割り込みをIOAPICが各LAPICに送信する。
xv6ではIOAPICのIOリダイレクションテーブルのエントリに割り込み先プロセッサのIDを設定し、IRQに応じて特定のプロセッサにリダイレクトする。  
内部からの割り込みは、LAPICがLVT（ローカルベクタテーブル）を使用してプロセッサに割り込みを送信する。
ローカル割り込みのソースとしては、プロセッサの割り込みピン（LINT0とLINT1ピン）、APICタイマ、温度センサ、パフォーマンスモニタリングカウンタ、他のプロセッサがある。  
ローカルAPICは多くのレジスタを備えており、そのアドレスと名前はIntel-SDMの「Table 10-1 Local APIC Register Address Map」にリストされている。
各レジスタへのアクセスは[「5.4. mpinit関数」](/chapter_05/05_04_mpinit.md)にてLAPICへのアクセスアドレスを設定した大域変数lapicを通して行う。
レジスタのアドレスをオフセットとして使うことにより、lapic変数から任意のレジスタにアクセスできる。
lapic変数はuint\*型なので、バイト単位でオフセットを使用するために4で割る。
例えばLocal APIC ID Register（0xFEE00020）にアクセスする際は、`lapic[0x0020 / 4]`とする。  

LAPICのレジスタへの書き込みにはlapicw関数を使用する。  
オフセット（index）で示されるレジスタに値（value）を書き込む。  
書き込み完了を待機するために、Local APIC ID Registerの読み込みを行う。

lapic.c
```c
// Local APIC registers, divided by 4 for use as uint[] indices.
#define ID      (0x0020/4)   // ID

/* 略 */

static void
lapicw(int index, int value)
{
  lapic[index] = value;
  lapic[ID];  // wait for write to finish, by reading
}
```

lapicinit関数は設定項目が多いので、設定項目毎に見ていくことにする。

lapic.c
```c
void
lapicinit(void)
{
  if(!lapic)
    return;

  // Enable local APIC; set spurious interrupt vector.
  lapicw(SVR, ENABLE | (T_IRQ0 + IRQ_SPURIOUS));

  // The timer repeatedly counts down at bus frequency
  // from lapic[TICR] and then issues an interrupt.
  // If xv6 cared more about precise timekeeping,
  // TICR would be calibrated using an external time source.
  lapicw(TDCR, X1);
  lapicw(TIMER, PERIODIC | (T_IRQ0 + IRQ_TIMER));
  lapicw(TICR, 10000000);

  // Disable logical interrupt lines.
  lapicw(LINT0, MASKED);
  lapicw(LINT1, MASKED);

  // Disable performance counter overflow interrupts
  // on machines that provide that interrupt entry.
  if(((lapic[VER]>>16) & 0xFF) >= 4)
    lapicw(PCINT, MASKED);

  // Map error interrupt to IRQ_ERROR.
  lapicw(ERROR, T_IRQ0 + IRQ_ERROR);

  // Clear error status register (requires back-to-back writes).
  lapicw(ESR, 0);
  lapicw(ESR, 0);

  // Ack any outstanding interrupts.
  lapicw(EOI, 0);

  // Send an Init Level De-Assert to synchronise arbitration ID's.
  lapicw(ICRHI, 0);
  lapicw(ICRLO, BCAST | INIT | LEVEL);
  while(lapic[ICRLO] & DELIVS)
    ;

  // Enable interrupts on the APIC (but not on the processor).
  lapicw(TPR, 0);
}
```

## LAPICの有効化と、スプリアス割り込みベクタの設定
LAPICは電源投入時やリセット時に無効化されているため、ここで有効化する。  
方法は2通りあり、Intel-SDMの「Vol.3 10.4.3 Enabling or Disabling the Local APIC」に記されている。
ここでは2番目のスプリアス割り込みベクタレジスタのフラグを使用する方法をとっている。  
レジスタにはアドレス0xFEE000F0（オフセット0xF0）でアクセスでき、各bitの意味はIntel-SDMの「Vol.3 10.9 SPURIOUS INTERRUPT」に記載されている。
APIC Software Enable/Disableビット（8bit）をセットするとLAPICが有効になる。  

LAPICの有効化と同時に、スプリアス割り込みに何番のIRQを使用するかを設定する。  
スプリアス割り込みは、電気的な干渉やデバイスの誤動作等により期せずして発生した割り込みのことで、スプリアス割り込みベクタレジスタの0～7bitでその際のIRQを設定できる。
ここでは63番をスプリアス割り込みに割り当てている。  
教科書[「xv6 a simple, Unix-like teaching operating system」（リンク1）](https://pdos.csail.mit.edu/6.828/2018/xv6/book-rev11.pdf)の「Code: Assembly trap handlers」によると、x86には256個の割り込みベクタ番号があり、0～31番は除算エラーや無効なアドレスのアクセスによるエラー等に割り当てられていて、32～255番はユーザが定義できるようになっている。
xv6は32～63番をハードウェア割り込みに使用する。システムコールは64番を使用する。
0～31番までの割り込みはIntel-SDMの「Vol.3 Table 6-1. Protected-Mode Exceptions and Interrupts」にリストしてある。
また、traps.hには割り込み番号の一覧が定義されている。

traps.h
```c
#define T_IRQ0          32      // IRQ 0 corresponds to int T_IRQ

/* 略 */

#define IRQ_SPURIOUS    31
```

lapic.c
```c
#define SVR     (0x00F0/4)   // Spurious Interrupt Vector
  #define ENABLE     0x00000100   // Unit Enable

/* 略 */

  lapicw(SVR, ENABLE | (T_IRQ0 + IRQ_SPURIOUS));
```

## LAPICタイマの設定
Intel-SDMの「Vol.3 10.5.4 APIC Timer」によると、LAPICには32bitプログラマブルタイマが含まれていて、Divide Configurationレジスタ、Initial Countレジスタ、Current Countレジスタ、LVT Timerレジスタの4つで設定を行う。  
動作としては、Initial Countレジスタから値がCurrent Countレジスタにコピーされて、後者がカウントダウンする。
カウントが0になると、タイマ割り込みが発生する。
ここでは以下の4つの設定を行う。
  - タイマの周波数
  - タイマの動作モードと割り込み番号
  - タイマの初期値

タイマの周波数:  
Divide Configurationレジスタ（0x3E0）に0xBを設定する。
このレジスタはその値によって、LAPICタイマの周波数が、プロセッサのバスクロックを何分の一したものにするかを設定する。
Intel-SDMに各bitパターンで設定されるの値が記されている。0b1011（0xB）の場合は1分の1が設定されるので、バスクロックと等しい周波数で動作する。

タイマの動作モードと割り込み番号:
LVT Timerレジスタ（0x320）に0x20020を設定する。
LAPICタイマの動作モードは、このレジスタの18bitと17bitによって決定される。ここではPeriodicモードが設定される。  
動作モードは以下の3種類。0b11は予約済み。  

| bit  | モード              | 動作                                                                                       |
|------|------------------|------------------------------------------------------------------------------------------|
| 0b00 | One-shot      | Current Countレジスタの値が0になり、割り込み発生後も0のまま変化しない。                                              |
| 0b01 | Periodic      | Current Countレジスタの値が0になり、割り込み発生後、Initial Countレジスタの値を再びコピーし、継続的にカウントダウンを行う。              |
| 0b10 | TSC-Deadline | Initial Countレジスタが無視され、Current Countレジスタは常に0になり、タイマの動作はIA32\_TSC\_DEADLINE MSRによって制御される。 |


割り込み番号は32番をタイマ割り込みに割り当てている。  

タイマの初期値:  
Initial Countレジスタ（0x380）に10000000を設定する。

まとめると、LAPICタイマは、10000000からバスクロックが進むごとにカウントダウンされ、0になるとIRQ32番で割り込みをかける。その後再び10000000からカウントダウンが始まる。


traps.h
```c
#define T_IRQ0          32      // IRQ 0 corresponds to int T_IRQ

#define IRQ_TIMER        0
```

lapic.c
```c
#define TIMER   (0x0320/4)   // Local Vector Table 0 (TIMER)
  #define X1         0x0000000B   // divide counts by 1
  #define PERIODIC   0x00020000   // Periodic

/* 略 */

#define TICR    (0x0380/4)   // Timer Initial Count
#define TDCR    (0x03E0/4)   // Timer Divide Configuration

/* 略 */

  lapicw(TDCR, X1);
  lapicw(TIMER, PERIODIC | (T_IRQ0 + IRQ_TIMER));
  lapicw(TICR, 10000000);
```

## LINT0ピンとLINT1ピンの無効化
Intel-SDMの「Vol.3 6.3.1 External Interrupts」によると、LINT0ピンとLINT1ピンはLAPICが有効な場合、割り込みに関与するようプログラムすることができる。
xv6では使用しないので無効にするのだと思う。
LINT0ピン（0x350）と、LINT1ピン（0x360）に0x00010000を設定している。
Intel-SDMの「Vol.3 Figure 10-8. Local Vector Table (LVT)」を見ると、16bitはマスクビットで1がMaskedとなっている。

lapic.c
```c
#define LINT0   (0x0350/4)   // Local Vector Table 1 (LINT0)
#define LINT1   (0x0360/4)   // Local Vector Table 2 (LINT1)

/* 略 */

  #define MASKED     0x00010000   // Interrupt masked

/* 略 */

  lapicw(LINT0, MASKED);
  lapicw(LINT1, MASKED);
```

## パフォーマンスモニタリングカウンタの無効化
パフォーマンスモニタリングカウンタが実装されている場合に、それを無効化する。  
Local APIC Versionレジスタ（0x30）のMax LVT Entry（16～23bit）の値が4以上であれば、実装されている。
この値の意味は、Intel-SDMの「Vol.3 10.4.8 Local APIC Version Register」に記載してあり、LVTエントリの数-1の値が設定されている。
LVTのエントリは同マニュアルの「Vol.3 Figure 10-8 Local Vector Table (LVT)」で確認でき、5番目がパフォーマンスモニタリングカウンタになっている。
なので、4（5-1=4）以上であればパフォーマンスモニタリングカウンタが実装されている。  
実装されている場合は、パフォーマンスモニタリングカウンタ（0x340）のマスクビットを立てて無効化する。

lapic.c
```c
#define VER     (0x0030/4)   // Version

/* 略 */

#define PCINT   (0x0340/4)   // Performance Counter LVT

/* 略 */

  #define MASKED     0x00010000   // Interrupt masked

/* 略 */

  if(((lapic[VER]>>16) & 0xFF) >= 4)
    lapicw(PCINT, MASKED);
```

##LAPICでエラーが生じた際のIRQ番号の設定
LVT Errorレジスタ（0x370）に51を設定する。  
このレジスタはIntel-SDMの「Vol.3 10.5.3 Error Handling」によると、LAPICでエラーが生じた際に割り込みを行うIRQ番号を設定することができる。
ここでは51番を設定している。

traps.h
```c
#define T_IRQ0          32      // IRQ 0 corresponds to int T_IRQ

/* 略 */

#define IRQ_ERROR       19
```

lapic.c
```c
#define ERROR   (0x0370/4)   // Local Vector Table 3 (ERROR)

/* 略 */

  lapicw(ERROR, T_IRQ0 + IRQ_ERROR);
```

## ESRをリセット
ESR（Error Status Register）（0x280）に0を設定する。
Intel-SDMの「Vol.3 10.5.3 Error Handling」によると、ESRはLAPICの割り込みでエラーが生じた際に検出されたエラーが書き込まれる。
ソースコード内のコメントによるとESRを初期化するためには0を2回書き込む必要があるらしいが、理由は見つけられなかった。

lapic.c
```c
#define ESR     (0x0280/4)   // Error Status

/* 略 */

  // Clear error status register (requires back-to-back writes).
  lapicw(ESR, 0);
  lapicw(ESR, 0);
```

## EOIレジスタをリセット
EOI（End Of Interrupt）レジスタ（0xB0）に0を設定する。  
Intel-SDMの「Vol.3 10.8.5 Signaling Interrupt Servicing Completion」でこのレジスタについて記載されている。
割り込みハンドラは、割り込み処理の終了時、iret命令の前にEOIレジスタに書き込みを行う必要がある。

lapic.c
```c
#define EOI     (0x00B0/4)   // EOI

/* 略 */

  lapicw(EOI, 0);
```

## ICRの設定
ICR（Interrupt Command Register）は64bitなので、LAPICアクセスアドレスのインデックスが32bitずつ0x300と0x310に分かれている。  
ICRについては、Intel-SDMの「Vol.3 10.6.1 Interrupt Command Register (ICR)」に記載があり、プロセッサ間割り込みIPI（Inter Processor Interrupts）の送信と設定を行う。各bitの意味は同マニュアルに記載されている。  
ここではDestination Shorthand（18, 19bit）とDelivery Mode（8～10bit）、Trigger Mode（15bit）を以下のように設定する。  
Destination Shorthand: All Including Self（0b01）。IPIの送信先を自分を含む全てのプロセッサにする。  
Delivery Mode: INITモード（0b101）。全てのプロセッサにINIT IPIを送信する。  
Trigger Mode: レベルモード（0b1）。INIT level de-assert delivery modeのトリガをレベルに設定する。  

上記IPIの送信が終了するまでwhileループする。  
ICRのDelivery Status（12bit）が1の時は送信が未完了、0になると送信完了となる。

Intel-SDMの「Vol.3 8.4.4.2 Typical AP Initialization Sequence」によると、起動後、各APはCLI命令とHLT命令が実行された状態になっており、BSPからのINIT IPIを待っている。  
同マニュアル「Vol.3 10.6.1 Interrupt Command Register（ICR）」によると、ここではINIT IPIをlevel de-assertで送信することにより、IPIの処理順序を決定するために使われるArb ID（Arbitration ID）をLAPIC IDと同じ値にしている。  
また、[「MultiProcessor Specification Version 1.4」（リンク14）](https://pdos.csail.mit.edu/6.828/2008/readings/ia32/MPspec.pdf)の「5.2 Integrated APIC Configurations」によると、APはRESETやINITシグナルを受けると、HALT状態になり、OSがSTARTUP IPIを送信するまで停止したままとなる。

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

  lapicw(ICRHI, 0);
  lapicw(ICRLO, BCAST | INIT | LEVEL);
  while(lapic[ICRLO] & DELIVS)
    ;
```

## TPRの設定
lapicinit関数の最後に、TPR（Task Priority Register）に0を設定する。  
TPRについてはIntel-SDMの「Vol.3 10.8.3.1 Task and Processor Priorities」に記載されており、名前の通りタスク優先度の設定を行うことができる。

lapic.c
```c
#define TPR     (0x0080/4)   // Task Priority

/* 略 */

  lapicw(TPR, 0);
```
