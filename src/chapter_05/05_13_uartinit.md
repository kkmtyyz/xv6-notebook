# 5.13. uartinit関数
この関数はシリアルポートがある場合はそれを初期化し、シリアルポートを通して画面に「xv6...」と表示する。

シリアルポートへのアクセスは[「XT, AT and PS/2 I/O port addresses」（リンク11）](http://bochs.sourceforge.net/techspec/PORTS.LST)によると、0x3f8～0x3ffで行える。  
個々のレジスタの役割や設定値に関しては[「Serial Programming/8250 UART Programming」（リンク28）](https://en.wikibooks.org/wiki/Serial_Programming/8250_UART_Programming)に詳しく記載がある。
UARTは12のレジスタに8つのポートからアクセスする。
各レジスタへのアクセス方法はリンク28の表「UART Registers」にまとまっている。
ポートへの読み書きとDivisor Latch Access Bit（DLAB）の状態によって切り替わる。

uart.cではシリアルポートの有無を示すフラグとして、int型の静的変数uartが定義されている。
この値が1のときシリアルポートが有る。
また、uart.cでは定数COM1が値0x3f8で定義されている。

まずUARTの設定は次の通り。
- **FIFO:** FIFOコントロールレジスタ（+2）に0を設定し、FIFOを無効化する。
リンク28の「FIFO Control Register」を読むと、uartの入出力はFIFOの形でバッファできると書いてあるが、xv6では使用しない。
- **DLAB:** ラインコントロールレジスタ（+3）に0x80を設定し、DLABを1にする。
- **ボーレート:** Divisor Latch Low Byteレジスタ（+0, DLABは1）に12を設定し、ボーレートを9600に設定する。
UARTチップは一般的に115.2kHzで動作するクロックを持っており、ここではボーレート9600で送信したいので12を設定する。
各ボーレートと設定すべき値に関してはリンク28の表「Divisor Latch Byte Values (common baud rates)」にまとまっている。
表によるとDivisor Latch Hight Byteレジスタ（+1）には0を設定しなければならないため、そのようにする。
- **ワードサイズ:** ラインコントロールレジスタ（+3）に0x03を設定し、DLABを0にするとともに、ワードサイズを8bitに設定する。
ワードサイズの設定はラインコントロールレジスタの0bitと1bitの組み合わせにより、5bitから8bitまで設定できる。
- **フロー制御:** モデムコントロールレジスタ（+4）に0を設定する。
このレジスタにより、ソフトウェアでハードウェアのフロー制御を行えるが、使用しないので0で初期化する。

シリアルポートの有無を確認する。
有無というより、UARTを使用してシリアルデータ通信が可能か否かを確認している。  
ラインステータスレジスタ（+5）の値が0xffと等しくない場合、シリアルポートが使用可能であることを示す。
スレジスタの各bitの役割についてはリンク28の「Line Status Register」に詳しく載っている。
特段0xffの場合に使用できないというようなことはないようだが、全てのbitが立っている状況が普通ではないことは分かる。
特にbit2（Parity Error）やbit3（Framing Error）が1になっていては通信はできない。

シリアルポートが使用できる場合、割り込みの設定を行う。  
割り込み識別レジスタ（+2）を読み、以前の割り込みに関する情報をクリアする。  
レシーバーバッファレジスタ（+0, DLABは0）を読み、バッファ内のデータをクリアする。  
[ioapicenable関数](https://kkmtyyz.github.io/xv6-notebook/chapter_05/05_09_consoleinit.html#ioapicenable関数)でシリアルポート（COM1）からの割り込みの設定と有効化をする。
IOAPICのリダイレクトテーブルにIRQ4番からの割り込みをIRQ36番としてBSP（cpu0）のLAPICにリダイレクトするよう設定する。

最後に「xv6...」という文字列をuartputc関数を使用して1文字ずつ画面に表示する。

uart.c
```c
#define COM1    0x3f8

static int uart;    // is there a uart?

void
uartinit(void)
{
  char *p;

  // Turn off the FIFO
  outb(COM1+2, 0);

  // 9600 baud, 8 data bits, 1 stop bit, parity off.
  outb(COM1+3, 0x80);    // Unlock divisor
  outb(COM1+0, 115200/9600);
  outb(COM1+1, 0);
  outb(COM1+3, 0x03);    // Lock divisor, 8 data bits.
  outb(COM1+4, 0);
  outb(COM1+1, 0x01);    // Enable receive interrupts.

  // If status is 0xFF, no serial port.
  if(inb(COM1+5) == 0xFF)
    return;
  uart = 1;

  // Acknowledge pre-existing interrupt conditions;
  // enable interrupts.
  inb(COM1+2);
  inb(COM1+0);
  ioapicenable(IRQ_COM1, 0);

  // Announce that we're here.
  for(p="xv6...\n"; *p; p++)
    uartputc(*p);
}
```

traps.h
```c
#define IRQ_COM1         4
```


## uartgetc関数
UARTのラインステータスレジスタ（+5）のData Ready（0bit）を見て、データの準備ができていることを確認する。  
レシーババッファレジスタ（+0, DLABは0）から入力されたデータを呼び出し元に返す。
シリアルポートではasciiコードがやり取りされるので、特に変換処理等はない。

uart.c
```c
#define COM1    0x3f8

/* 略 */

static int
uartgetc(void)
{
  if(!uart)
    return -1;
  if(!(inb(COM1+5) & 0x01))
    return -1;
  return inb(COM1+0);
}
```


## uartputc関数
uartputc関数は、シリアルポートに文字を書き込む。

forループでUARTのラインステータスレジスタ（+5）のEmpty Transmitter Holding Register（5bit）が0になるまで、最大128回ループする。
5bitは1の場合は送信中、0の場合は送信可能。  
microdelay関数はlapic.cに定義されているが実体は何もしない。
コメントに、実際のハードウェアではマイクロ秒単位のスピンロックを動的に行うと書いてある。  
Transmitter Holding Buffer（+0, DLABは0）に引数の文字を書き込む。

uart.c
```c
void
uartputc(int c)
{
  int i;

  if(!uart)
    return;
  for(i = 0; i < 128 && !(inb(COM1+5) & 0x20); i++)
    microdelay(10);
  outb(COM1+0, c);
}
```

lapic.c
```c
// Spin for a given number of microseconds.
// On real hardware would want to tune this dynamically.
void
microdelay(int us)
{
}
```

