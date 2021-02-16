# 5.8. ioapicinit関数
この関数は、大域変数ioapicにIOAPICへのアクセスアドレスを設定し、IOリダイレクションテーブルの設定を行う。  
IRQ0～23番が、BSPにIRQ32～55番でリダイレクトされるよう設定を行い、それら全てを無効化する。

[「OSDev IOAPIC」（リンク17）](https://wiki.osdev.org/IOAPIC)のexternal linksからIOAPICの仕様書[「82093AA I/O ADVANCED PROGRAMMABLE INTERRUPT CONTROLLER (IOAPIC)」（リンク18）](http://web.archive.org/web/20161130153145/http://download.intel.com/design/chipsets/datashts/29056601.pdf)が確認できる。  
仕様書の「3.0. REGISTER DESCRIPTION」によると、IOAPICの操作はI/O Register Selectレジスタにアクセスしたいレジスタのオフセットを書き込み、I/O Windowレジスタを通してデータの読み書きを行う。
I/O Register Selectレジスタのアドレスはデフォルトで0xFEC00000だが、制限された範囲内で変更が可能。xv6ではデフォルトのまま使用する。  
I/O Windowsレジスタのアドレスもデフォルトの0xFEC00010になる。  
xv6ではこの2つのレジスタにアクセスするため、ioapic構造体を定義している。
2つのアドレスは16バイト離れているため、padフィールドを設けている。
レジスタへの読み書きはioapicread関数とioapicwrite関数で行う。

ioapic.c
```c
#define IOAPIC  0xFEC00000   // Default physical address of IO APIC

/* 略 */

volatile struct ioapic *ioapic;

// IO APIC MMIO structure: write reg, then read or write data.
struct ioapic {
  uint reg;
  uint pad[3];
  uint data;
};

static uint
ioapicread(int reg)
{
  ioapic->reg = reg;
  return ioapic->data;
}

static void
ioapicwrite(int reg, uint data)
{
  ioapic->reg = reg;
  ioapic->data = data;
}
```

仕様書の「Table 2. IOAPIC Registers」より、アクセスできるIOAPICのレジスタとオフセットは以下の通り。
- IOAPIC ID（オフセット0x00）
- IOAPIC Version（0x01）
- IOAPIC Arbitration ID（0x02）
- Redirection Table（0x10～0x3F）

IOリダイレクションテーブルを使用することで、外部割込みソースからの割込みをそのIRQ番号に応じてLAPICにリダイレクトすることができる。
テーブルのインデックスが入力されるIRQ番号に対応しており、各エントリにリダイレクト先LAPICやIRQ番号等を設定する。  
IOAPICへ入力されるIRQ番号がそれぞれどの割込みソースに割り当てられているのかは、そのデバイスが接続されているIOAPICのピンに依存する。
しかし、一般的に使用されるIRQ番号は決まっており、仕様書の「2.4. Interrupt Signals」の記載のほか、[「OSDev Interrupt」（リンク29）](https://wiki.osdev.org/Interrupts)の「General IBM-PC Compativle Interrupt Information」にもリストされている。
これらによると、例えばIRQ0はProgrammable Interrupt Timer Interruptに、IRQ1はKeyboard Interruptに使用されることがわかる。  
エントリのサイズは64bitで、上下32bitずつオフセットを変えてアクセスすることになる。
仕様書の表2によるとテーブルのオフセット範囲は0x10～0x3Fであるため、エントリ数は24だが、IOAPIC VersionレジスタのMaximum Redirection Entryフィールド（16～23bit）から最大エントリ番号を得ることができる。
記載によると、最大で240エントリ持つことができる。

ioapicinit関数では、まず大域変数ioapicにIOAPICへのアクセスアドレス0xFEC00000を設定する。  
次にIOリダイレクションテーブルの最大エントリ番号と、IOAPIC IDを取得する。
デバッガで最大エントリ番号を見ると、今回は仕様書の値と同じ24（値は23）だった。
IOAPIC IDが[「5.4. mpinit関数」](/chapter_05/05_04_mpinit.md)にてMP設定テーブルから取得したIOAPIC IDと異なる場合、コンソールにメッセージを出力する。
出力に使用するcprintf関数は[「5.17 startothers関数」](/chapter_05/05_19_startothers.md#cprintf関数)で見る。

for文でIOリダイレクションテーブルのエントリを走査し、以下の設定を行う。
エントリの構造は仕様書の「3.2.4. IOREDTBL[23:0]—I/O REDIRECTION TABLE REGISTERS」に記載されている。  
- Interrupt Mask（16bit）に1をセットしエントリを無効化
- Interrupt Vector（0～7bit）にリダイレクト先IRQ番号としてタイマー割込み32番（T\_IRQ0）から順にセット
- Destination Mode（11bit）を0にしてPhysical Modeをセット
- Destination Field（56～63bit）に0をセット。このフィールドはDestination Modeによって意味が異なり、Physical Modeではリダイレクト先のLAPIC IDを意味する。0なのでBSPにリダイレクトする。

ioapic.c
```c
#define REG_ID     0x00  // Register index: ID
#define REG_VER    0x01  // Register index: version
#define REG_TABLE  0x10  // Redirection table base

/* 略 */

#define INT_DISABLED   0x00010000  // Interrupt disabled

/* 略 */

void
ioapicinit(void)
{
  int i, id, maxintr;

  ioapic = (volatile struct ioapic*)IOAPIC;
  maxintr = (ioapicread(REG_VER) >> 16) & 0xFF;
  id = ioapicread(REG_ID) >> 24;
  if(id != ioapicid)
    cprintf("ioapicinit: id isn't equal to ioapicid; not a MP\n");

  // Mark all interrupts edge-triggered, active high, disabled,
  // and not routed to any CPUs.
  for(i = 0; i <= maxintr; i++){
    ioapicwrite(REG_TABLE+2*i, INT_DISABLED | (T_IRQ0 + i));
    ioapicwrite(REG_TABLE+2*i+1, 0);
  }
}
```

このIOリダイレクションテーブルの設定により、IRQ0～23番がBSPのLAPICにIRQ32～55番でリダイレクトされるが、この時点では全てのリダイレクトが無効化された状態になっている。  
エントリの有効化は、後ほどioapicenable関数を呼び出して個別に行う。その際に割込み先LAPIC IDも再設定する。
