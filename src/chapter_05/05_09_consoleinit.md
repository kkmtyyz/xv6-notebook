# 5.9. consoleinit関数
この関数はdevsw配列の1番にコンソール読み書き用の関数を設定し、IOAPICのキーボードコントローラからの割込みのリダイレクト（IRQ1からIRQ33へのリダイレクト）を有効化する。

デバイスの読み書きにはdevsw構造体を使用する。
この構造体は関数ポインタreadとwriteを持っており、そこにデバイス毎の読み書き用の関数を持っておく。  
file.cでdevsw構造体の配列devswが定義されており、デバイス10個分の構造体を持つことができる。
この配列の1番目をコンソールのために使用する。残りの9個は使用しない。
ディスクの読み書きにはdevsw構造体を使用せず、ide.cに定義されているiderw関数を使用する。

file.h
```c
struct devsw {
  int (*read)(struct inode*, char*, int);
  int (*write)(struct inode*, char*, int);
};
```

file.c
```c
struct devsw devsw[NDEV];
```

param.h
```c
#define NDEV         10  // maximum major device number
```


コンソールの排他制御にはconsole.cの静的変数consを使用する。
consはspinlock構造体のlockフィールドと、int型のlockingフィールドを持っている。
lockフィールドはコンソールを使用する際のロックに使われる。
ロックについては[「5.10. ロック（spinlock, sleeplock）」](/chapter_05/05_10_lock.md)に書く。
lockingフィールドの値は通常1で、panic関数が呼ばれると0になり、その後1に戻ることはない。
このフィールドはcprintf関数内で使用されており、値が1の場合はコンソールのロックを取り、0の場合は取らない。
おそらくpanic関数が呼ばれた際に、既にコンソールのロックが取られている可能性があるため、panic関数でlockingを0にするのだと思う。

console.c
```c
static struct {
  struct spinlock lock;
  int locking;
} cons;
```


consoleinit関数ではまずconsのlockフィールドを初期化する。  
devsw配列のコンソール（1番）のwriteフィールドにconsolewrite関数を設定し、readフィールドにconsoleread関数を設定する。  
次にconsのlockingフィールドを1で初期化する。
この値は上述したように、panicが呼ばれるまで0にならない。  
最後に、ioapicenable関数でキーボードコントローラからの割込みを有効化する。

file.h
```c
#define CONSOLE 1
```

traps.h
```c
#define IRQ_KBD          1
```

console.c
```c
void
consoleinit(void)
{
  initlock(&cons.lock, "console");

  devsw[CONSOLE].write = consolewrite;
  devsw[CONSOLE].read = consoleread;
  cons.locking = 1;

  ioapicenable(IRQ_KBD, 0);
}
```

## consolewrite関数
この関数はコンソールに文字を出力する。

inodeのロックを解放し、コンソールのロックを取ってbufからn文字分出力する。
出力が終わったらコンソールのロックを解放し、inodeのロックを再び取得する。
コンソール出力中はinodeのロックが不要なのでロックを解放する。
inodeについては[「5.11. iノードのロック」](chapter_05/05_11_ilock.md)に書く。  
出力はconsputc関数で行う。
consputc関数の引数はint型だが、アスキーコードの範囲で渡したいので、0xffで論理積を取って1バイトだけ渡す。

console.c
```c
int
consolewrite(struct inode *ip, char *buf, int n)
{
  int i;

  iunlock(ip);
  acquire(&cons.lock);
  for(i = 0; i < n; i++)
    consputc(buf[i] & 0xff);
  release(&cons.lock);
  ilock(ip);

  return n;
}
```

consputc関数はシリアルポートが有ればそちらに書き込みを行い、無ければメモリにマップされているビデオメモリに書き込む。
シリアルポートへの書き込みはuartputc関数で行い、ビデオメモリへの書き込みはcgaputc関数で行う。
シリアルポートの有無はmain関数から呼ばれるuartinit関数で確認する。
また、シリアルポートにバックスペースを書き込む際には、'\b'でカーソル位置を1文字戻し、空白を印字し、再びカーソル位置を1文字戻す。  
console.cにはint型の静的変数panickedが定義されており、この値はpanic関数が呼ばれると1になる。
consputc関数は、panic関数が呼ばれている場合は、割り込みを無効化し、無限ループする。

console.c
```c
static int panicked = 0;

/* 略 */

void
consputc(int c)
{
  if(panicked){
    cli();
    for(;;)
      ;
  }

  if(c == BACKSPACE){
    uartputc('\b'); uartputc(' '); uartputc('\b');
  } else
    uartputc(c);
  cgaputc(c);
}
```

uartputc関数は[「5.13. uartinit」](/chapter_05/05_13_uartinit.md)に書くとして、cgaputc関数を見る。
cgaputc関数はメモリ上のビデオメモリ領域に書き込むことで画面出力を行う。

カーソル位置はCRTコントローラを使用して読み書きする。
[「Hardware Level VGA and SVGA Video Programming Information Page CRT Controller Registers」（リンク19）](http://web.stanford.edu/class/cs140/projects/pintos/specs/freevga/vga/crtcreg.htm)
によると、CRTCレジスタにはアドレスレジスタとデータレジスタがあり、それぞれポート0x3d4とポート0x3d5からアクセスできる。
リンク先の「Cursor Location High Register (Index 0Eh)」と「Cursor Location Low Register (Index 0Fh)」に、アドレスレジスタに14（0xE）と15（0xF）を書き込むことで、データレジスタからカーソル位置を取得することができると記載がある。
カーソル位置は画面左上を0として、1行80文字、25行までで連続して増加していく。

プロテクトモード時のビデオモードに関しては[「OSDev Drawing In Protected Mode」（リンク20）](https://wiki.osdev.org/Drawing_In_Protected_Mode)に記載されており、ここではCGAモードなので物理アドレス0xb8000からビデオメモリ領域になっている。  
ビデオメモリへは2バイトずつ書き込みを行い、書き込む値は[「OSDev Text UI」（リンク21）](https://wiki.osdev.org/Text_UI)によると、上位1バイトで色を指定し、下位1バイトでASCIIコードを指定する。
色の指定方法はリンク先の「Colours」で図解されており、上位4bitで背景色、下位4bitで文字色を指定する。
また、3, 7bit目がそれぞれbright bitとなっており、立てると明るい色になる。
cgaputc関数では0x0700で論理和を取り文字色をライトグレイにしているが、例えば0xb200で論理和を取ると背景色がライトシアンとなり、文字色が緑色となる。

cgaputc関数ではint型の変数posでカーソル位置を操作する。  
引数が改行コードの場合、posが次の行の先頭になるように、80 - 現在位置%80だけ加算する。  
引数がバックスペースかつposが0より大きい場合、posをデクリメントする。  
それ以外の場合は現在位置の次の位置に引数の文字（1バイト）を文字色ライトグレイ、背景色黒で描画する。  
変数crtはメモリ上のビデオメモリ領域の開始アドレス（0xb8000）を持っている。  
カーソル位置が0未満或いは25行80列より大きいところを指している場合、panicする。  
25行目を指している場合、memmove関数を使用して全体を1行分前方に動かし、カーソル位置も1行分減らす。
全体を1行分前にコピーするので、最終行に値が残っている。それをmemset関数で0にリセットする。  
最後に画面上の現在の位置にスペースを書き込む。

console.c
```c
#define BACKSPACE 0x100
#define CRTPORT 0x3d4
static ushort *crt = (ushort*)P2V(0xb8000);  // CGA memory

static void
cgaputc(int c)
{
  int pos;

  // Cursor position: col + 80*row.
  outb(CRTPORT, 14);
  pos = inb(CRTPORT+1) << 8;
  outb(CRTPORT, 15);
  pos |= inb(CRTPORT+1);

  if(c == '\n')
    pos += 80 - pos%80;
  else if(c == BACKSPACE){
    if(pos > 0) --pos;
  } else
    crt[pos++] = (c&0xff) | 0x0700;  // black on white

  if(pos < 0 || pos > 25*80)
    panic("pos under/overflow");

  if((pos/80) >= 24){  // Scroll up.
    memmove(crt, crt+80, sizeof(crt[0])*23*80);
    pos -= 80;
    memset(crt+pos, 0, sizeof(crt[0])*(24*80 - pos));
  }

  outb(CRTPORT, 14);
  outb(CRTPORT+1, pos>>8);
  outb(CRTPORT, 15);
  outb(CRTPORT+1, pos);
  crt[pos] = ' ' | 0x0700;
}
```

## memmove関数
この関数は第二引数のアドレス（ソース）から第三引数のサイズ分、第一引数のアドレス（移動先）にコピーする。
このとき、ソースが移動先より手前で開始し、移動先がコピーサイズ分に被るとき、頭からコピーしていくとソースの一部が上書きされ失われるので、末尾からコピーしていく。

string.c
```c
void*
memmove(void *dst, const void *src, uint n)
{
  const char *s;
  char *d;

  s = src;
  d = dst;
  if(s < d && s + n > d){
    s += n;
    d += n;
    while(n-- > 0)
      *--d = *--s;
  } else
    while(n-- > 0)
      *d++ = *s++;

  return dst;
}
```

## consoleread関数
この関数はコンソールからnバイト読み込む。

コンソールへの入力に起因する割り込みから順に書くことにする。
なお、割り込み時の動作については[「」]()に書く。  
キーボードあるいはシリアルポートから入力が行われると、割込みが生じ最終的にkbdintr関数かuartintr関数が呼ばれる。
それらはいずれもconsoleintr関数を呼び出しており、引数としてkbdgetc関数あるいはuartgetc関数を渡す。
uartgetc関数については[「5.13. uartinit関数」](/chapter_05/05_13_uartinit.md)に書く。

kbd.c
```c
void
kbdintr(void)
{
  consoleintr(kbdgetc);
}
```

uart.c
```c
void
uartintr(void)
{
  consoleintr(uartgetc);
}
```

### kbdgetc関数
この関数はキーボードコントローラからの入力信号（スキャンコード）を解析し、キーボードに入力された文字コードを返す。  
[「Bochs Developers Guide」（リンク9）](http://bochs.sourceforge.net/doc/docbook/development/index.html)によると、ポート0x60（アウトプットバッファ）から、スキャンコードを取得できる。
スキャンコードに関しては[「Keyboard scancodes Andries Brouwer」（リンク7）](https://www.win.tue.nl/~aeb/linux/kbd/scancodes.html)で解説されており、キーが押されるときmakeコードが入力され、離されるときにbreakコードが入力される。
breakコードはmakeコードに0x80が加算された値になっている。
例えば、aキーが押されると0x1eが入力され、キーが離されて0x9eが入力される。
また、コード0xe0と0xe1はエスケープシーケンスになっており、プリフィクスとして使用される。
あるキーを押すと、0xe0が入力され、続けて入力される特定のスキャンコードと共に解釈することになる。
例えば、方向キー上が押されると0xe0が入力され、続けて0x48が入力され、離された後に0xc8が入力される。
なので、gdbで確認する際はキーが押されたときと、離されたときを別々に観測するとよい。

xv6では、スキャンコードから文字コードへの変換にkbd.hに定義されている3つの配列normalmap、shiftmap、ctlmapを使用する。  
各配列は名前の通り、普通の入力、シフトキーを使用した入力、コントロールキーを使用した入力で使用される。
参照する配列の切り替えには、shiftcodeとtogglecodeの2つの配列を使用する。
そこにはCTRLキーやALTキー、CAPSLOCKキー等が定義されている。  
kbdget関数では配列charcodeに変換に使用する3つの配列を持っておく。
4番目のctlmapはシフトキーとコントロールキーが同時に入力された場合に使用される。

ctlmap配列ではCマクロが使用されており、このマクロは与えられた文字コードを制御コードに変換する。
各制御コードは[「ASCII Code - The extended ASCII table」（リンク27）](https://www.ascii-code.com/)で確認できる。
例えば`C('C')`の場合、
`'C' - '@' = 0x43 - 0x40 = 0x03`
なので、End of Text（0x03）となる。

kbd.h
```c
// C('A') == Control-A
#define C(x) (x - '@')
```

エスケープシーケンス（0xe0）や、シフトキー、コントロールキー、CAPSLOCKのオンオフは変数shiftの各bitで表現する。
各bitの役割は以下。

kbd.h
```c
>  9 #define SHIFT           (1<<0)
> 10 #define CTL             (1<<1)
> 11 #define ALT             (1<<2)
> 12 
> 13 #define CAPSLOCK        (1<<3)
> 14 #define NUMLOCK         (1<<4)
> 15 #define SCROLLLOCK      (1<<5)
> 16 
> 17 #define E0ESC           (1<<6)
```

変数shiftのSHIFTビットもCTLビットも設定されていれば4エントリ目のctlmapテーブルが使用される。

kbdgetc関数ははじめに、キーボードコントローラのステータス（0x64）を取得する。
ステータスの0bitからインプットバッファの状態を確認でき、1のときフル、0のとき空を示す。  
アウトプットバッファ（0x60）から1つ分のスキャンコードを取得し、次のように分岐する。
- エスケープシーケンス（0xE0）の場合、変数shiftのE0ESCビットを立てる。
- breakコードの場合、つまりキーが離された場合、変数shiftのE0ESCビット、SHIFTビット、CTLビット、ALTビットを0にする。
CAPSLOCKビットやNUMLOCKビット、SCROLLLOCKビットは入力を跨いで有効となるため変更しない。
エスケープシーケンス以外の場合、0x7fとの論理積によりmakeコードを取得している。
- 変数shiftのE0ESCビットが1の場合、変数dataにbreakコード（0x80を論理和）を代入し、E0ESCビットを0にする。エスケープシーケンス（0xE0）が使用される場合breakコードが入力されないので、続くスキャンコードをbreakコードに変換する必要がある。

次に、スキャンコードの変換に用いる配列を決定するため、変数shiftを設定する。
shiftcode配列はそのままビットを設定するが、togglecode配列はCAPSLOCKやNUMLOCK等を扱うため排他的論理和でビットを更新する。  
最後に配列から文字コードを取得するが、変数shiftのCAPSLOCKビットが1の場合、アルファベットの大文字と小文字を変換する。

kbd.c
```c
int
kbdgetc(void)
{
  static uint shift;
  static uchar *charcode[4] = {
    normalmap, shiftmap, ctlmap, ctlmap
  };
  uint st, data, c;

  st = inb(KBSTATP);
  if((st & KBS_DIB) == 0)
    return -1;
  data = inb(KBDATAP);

  if(data == 0xE0){
    shift |= E0ESC;
    return 0;
  } else if(data & 0x80){
    // Key released
    data = (shift & E0ESC ? data : data & 0x7F);
    shift &= ~(shiftcode[data] | E0ESC);
    return 0;
  } else if(shift & E0ESC){
    // Last character was an E0 escape; or with 0x80
    data |= 0x80;
    shift &= ~E0ESC;
  }

  shift |= shiftcode[data];
  shift ^= togglecode[data];
  c = charcode[shift & (CTL | SHIFT)][data];
  if(shift & CAPSLOCK){
    if('a' <= c && c <= 'z')
      c += 'A' - 'a';
    else if('A' <= c && c <= 'Z')
      c += 'a' - 'A';
  }
  return c;
}
```

### consoleintr関数
この関数はキーボードやシリアルポートから入力された値を読み取り、それをコンソールに出力する。
また、入力を待っているプロセスを起床させ、値を渡す。

入力を待っているプロセスはconsoleread関数でスリープしており、input構造体を通して値を渡す。  
input構造体は次の4つのフィールドで構成される。
- buf: 入力された直近128文字を持つ
- r: bufフィールドからプロセスが読み出した位置を示すとともに、コンソールへの出力を待っているプロセスを起床させるチャネルとしても使用される。
- w: bufフィールドからコンソールに出力した位置を示す
- e: bufフィールドの現在位置を表す

bufは要素127まで来たら要素0から上書きするため、読み書きする際は要素数で割った余りをインデックスとして使用する。
r、w、eの各フィールドは0から開始する。
文字が入力される度にeがインクリメントされbufに文字が入る。
それをコンソールに出力するとwがeで更新され、追いつく。
コンソールへの文字出力後、consoleread関数でスリープしているプロセスが起床され、そこでrからwの位置までbufの中身を読み取る。
r、w、eは明示的に初期化されることはなく、uintの最大値2の32乗-1まで増加し続ける。

console.c
```c
#define INPUT_BUF 128
struct {
  char buf[INPUT_BUF];
  uint r;  // Read index
  uint w;  // Write index
  uint e;  // Edit index
} input;
```

while文で1文字ずつ読み、switch文で次のように分岐する。
- Ctrl + Pの場合、procdump関数を実行する。この関数はプロセスの一覧を出力し、スリープ状態のプロセスの場合はコールスタックも出力する。
- Ctrl + Uの場合、コンソールの現在の行の文字を全て消す。
現在位置eを行頭まで戻しつつ、文字を1文字ずつ消していく。
- Ctrl + H、あるいはバックスペース（0x7f）の場合、1文字消す。
バックスペースが入力されると、キーボードの場合はkbdgetc関数からバックスペース（0x08）が返ってきてCtrl + Hのケースに一致し、シリアルポートの場合はuartgetc関数からデリート（0x7f）が返ってきてこのケースに一致する。
コンソールの現在位置eと前回の最終出力位置wが等しくない場合、つまり消せる入力がある場合はeをデクリメントし、consputc関数を使ってバックスペースを出力する。
- 上記以外の場合、文字をコンソールに出力する。
キャリッジリターン（0x0d）は改行（0x0a）に変換する。

次の場合、出力済み位置wに現在位置eを代入し、rのアドレスをチャネルとしてスリープしているプロセスを起床させる。
- 入力が終了する（Enter）
- Ctrl + Dが入力される（EOT）
- 読み込む必要がある文字列がbufの容量と等しくなった場合

スリープしているプロセスはコンソールへ入力された文字列（bufの中身）を読み取り、解釈する。

console.c
```c
void
consoleintr(int (*getc)(void))
{
  int c, doprocdump = 0;

  acquire(&cons.lock);
  while((c = getc()) >= 0){
    switch(c){
    case C('P'):  // Process listing.
      // procdump() locks cons.lock indirectly; invoke later
      doprocdump = 1;
      break;
    case C('U'):  // Kill line.
      while(input.e != input.w &&
            input.buf[(input.e-1) % INPUT_BUF] != '\n'){
        input.e--;
        consputc(BACKSPACE);
      }
      break;
    case C('H'): case '\x7f':  // Backspace
      if(input.e != input.w){
        input.e--;
        consputc(BACKSPACE);
      }
      break;
    default:
      if(c != 0 && input.e-input.r < INPUT_BUF){
        c = (c == '\r') ? '\n' : c;
        input.buf[input.e++ % INPUT_BUF] = c;
        consputc(c);
        if(c == '\n' || c == C('D') || input.e == input.r+INPUT_BUF){
          input.w = input.e;
          wakeup(&input.r);
        }
      }
      break;
    }
  }
  release(&cons.lock);
  if(doprocdump) {
    procdump();  // now call procdump() wo. cons.lock held
  }
}
```

consoleread関数はinput構造体のbufフィールドを読み込む。  
コンソールの読み込み位置（r）が出力済み位置（w）に追いついている場合、入力待ちのために現在のプロセスをスリープ状態にする。
このため第一引数で与えられたiノードのロックを開放する。
スリープについては[「5.10. ロック（spinlock, sleeplock）」](/chapter_05/05_10_lock.md)に書く。  
プロセスが起床し、出力済み位置（w）が読み込み位置（r）よりも先に進んだ場合、rをインクリメントしbufから文字コードを取り出す。
文字コードがCtrl + D（EOT）の場合、読み込みのwhileループから抜ける。
このとき、既に何文字か読み込んでいた場合はrをデクリメントする。
これにより、次回実行時にはwhileループに入らず、そのまま同じEOTを読み込むことになり、直ぐにbreakでループを抜けることになる。
呼び出し元にreturnで返される読み込んだ文字数は0になり、第二引数dstには何も書き込まれず、呼び出し元は読み込みたい文字列がEOTで終了されたことを知ることができる。

console.c
```c
int
consoleread(struct inode *ip, char *dst, int n)
{
  uint target;
  int c;

  iunlock(ip);
  target = n;
  acquire(&cons.lock);
  while(n > 0){
    while(input.r == input.w){
      if(myproc()->killed){
        release(&cons.lock);
        ilock(ip);
        return -1;
      }
      sleep(&input.r, &cons.lock);
    }
    c = input.buf[input.r++ % INPUT_BUF];
    if(c == C('D')){  // EOF
      if(n < target){
        // Save ^D for next time, to make sure
        // caller gets a 0-byte result.
        input.r--;
      }
      break;
    }
    *dst++ = c;
    --n;
    if(c == '\n')
      break;
  }
  release(&cons.lock);
  ilock(ip);

  return target - n;
}
```

## ioapicenable関数
この関数は指定されたIRQのリダイレクトを有効化し、リダイレクト先LAPICを設定する。  

IOリダイレクションテーブルの設定は[「5.8. ioapicinit関数」](/chapter_05/05_08_ioapicinit.md)で行った。
ここではそこでも使用したioapicwrite関数にて、エントリのDestination Field（56bit（32+24））にLAPIC IDを設定する。
また、同時にInterrupt Mask（16bit）が0に設定されることにより割り込みのリダイレクトを有効化している。

ioapic.c
```c
void
ioapicenable(int irq, int cpunum)
{
  // Mark interrupt edge-triggered, active high,
  // enabled, and routed to the given cpunum,
  // which happens to be that cpu's APIC ID.
  ioapicwrite(REG_TABLE+2*irq, T_IRQ0 + irq);
  ioapicwrite(REG_TABLE+2*irq+1, cpunum << 24);
}
```

