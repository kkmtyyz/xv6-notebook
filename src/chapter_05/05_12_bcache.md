# 5.12. バッファキャッシュとディスクの読み書き
スケジューラの実行までに使用するバッファキャッシュ周りの操作について書く。

- [buf構造体](/chapter_05/05_12_bcache.md#buf構造体)
- [buf構造体の取得（bread関数）](/chapter_05/05_12_bcache.md#buf構造体の取得bread関数)
- [buf構造体の書き込み（bwrite関数）](/chapter_05/05_12_bcache.md#buf構造体の書き込みbwrite関数)
- [ディスクの読み書き（iderw関数）](/chapter_05/05_12_bcache.md#ディスクの読み書きiderw関数)
- [データブロックの割り当て（balloc関数）]()
- [データブロックの解放（bfree関数）]()


## buf構造体

バッファキャッシュについては、[「xv6 a simple, Unix-like teaching operating system」（リンク1）](https://pdos.csail.mit.edu/6.828/2018/xv6/book-rev11.pdf)の6章「File system」を読むとよい。

buf構造体はディスクから読み込んだ1ブロック分のデータをキャッシュしておくための構造体。  
30個分を大域変数bcacheのbufフィールドに持つ。
bcache構造体はbinit関数で初期化される。

バッファキャッシュは循環リストになっており、各buf構造体のnextフィールドとprevフィールドでリンクする。
アクセスする際にはbcache構造体のheadフィールドを基点として、nextかprevでどちらかに回って走査する。
直近使用されたバッファはnext側の先頭に移動されるため、欲しいデータが既にバッファされているか確認するときはnext側に回る。
逆にデータをバッファする際は、未使用のバッファを探すためにprev側から回る。  
buf構造体のflagsフィールドは0x0が未使用、B\_VALID（0x2）が使用中、B\_DIRTY（0x4）が変更済みを示す。  
dataフィールドに1ブロック分のデータを持つ。

bio.c
```c
struct {
  struct spinlock lock;
  struct buf buf[NBUF];

  // Linked list of all buffers, through prev/next.
  // head.next is most recently used.
  struct buf head;
} bcache;
```

buf.h
```c
struct buf {
  int flags;
  uint dev;
  uint blockno;
  struct sleeplock lock;
  uint refcnt;
  struct buf *prev; // LRU cache list
  struct buf *next;
  struct buf *qnext; // disk queue
  uchar data[BSIZE];
};
#define B_VALID 0x2  // buffer has been read from disk
#define B_DIRTY 0x4  // buffer needs to be written to disk
```


## buf構造体の取得（bread関数）

bread関数を使用して、欲しいブロックのbuf構造体を取得する。  
ブロックがバッファキャッシュに存在しない場合は、空のエントリにディスクからブロックを読み込む。
バッファキャッシュからの取得はbget関数で行い、ディスクからの読み込みは[iderw関数](/chapter_05/05_12_bcache.md#ディスクの読み書きiderw関数)で行う。

bio.c
```c
struct buf*
bread(uint dev, uint blockno)
{
  struct buf *b;

  b = bget(dev, blockno);
  if((b->flags & B_VALID) == 0) {
    iderw(b);
  }
  return b;
}
```

bget関数はデバイス番号とブロック番号を用いてバッファキャッシュからbuf構造体を探し、無ければ未使用のバッファを返す。  
最近キャッシュされた順に探すため、headからnext方向に走査する。
キャッシュを見つけた場合はbuf構造体の参照カウンタをインクリメントし、スリープロックを取る。
見つからなかった場合は未使用のbuf構造体を探すため、headからprev方向に走査する。  
もしもキャッシュが30個全て使用されていた場合はpanicする。

bio.c
```c
static struct buf*
bget(uint dev, uint blockno)
{
  struct buf *b;

  acquire(&bcache.lock);

  // Is the block already cached?
  for(b = bcache.head.next; b != &bcache.head; b = b->next){
    if(b->dev == dev && b->blockno == blockno){
      b->refcnt++;
      release(&bcache.lock);
      acquiresleep(&b->lock);
      return b;
    }
  }

  // Not cached; recycle an unused buffer.
  // Even if refcnt==0, B_DIRTY indicates a buffer is in use
  // because log.c has modified it but not yet committed it.
  for(b = bcache.head.prev; b != &bcache.head; b = b->prev){
    if(b->refcnt == 0 && (b->flags & B_DIRTY) == 0) {
      b->dev = dev;
      b->blockno = blockno;
      b->flags = 0;
      b->refcnt = 1;
      release(&bcache.lock);
      acquiresleep(&b->lock);
      return b;
    }
  }
  panic("bget: no buffers");
}
```


## buf構造体の書き込み（bwrite関数）
iderw関数でディスクへbuf構造体を書き出す。

void
```c
bwrite(struct buf *b)
{
  if(!holdingsleep(&b->lock))
    panic("bwrite");
  b->flags |= B_DIRTY;
  iderw(b);
}
```


## ディスクの読み書き（iderw関数）
buf構造体をidequeueにエンキューし、それがキューの先頭である場合にディスクの読み込み或いは書き込み処理を行う。


buf構造体のデバイス番号が0でなかった場合、かつhavedisk1フラグが0だった場合はpanicする。
ディスクはカーネルイメージのディスク0と、ファイルシステムイメージのディスク1の2つのみであるため、デバイス番号は0か1になる。
また、デバイス番号が1ならば、ファイルシステムイメージの存在が確認されている必要があり、havedisk1フラグでそれを確認できる。
このフラグはideinit関数でディスクをチェックする際に設定される。

ideへの要求となるbuf構造体はide.cで定義されているidequeueにエンキューされ、idestart関数でそれをひとつずつideに処理してもらう。
iderw関数では、もしも引数のbuf構造体がidequeueの先頭だった場合、idestart関数でideに処理要求を出す。
ideの読み書きが終わるまではスリープ状態となり、割り込みを待つ。
ideの処理が終わると、割り込みによりideintr関数が実行され、その中でidequeueからデキューされ、順次idestart関数が呼ばれる。

ide.c
```c
static struct buf *idequeue;

/* 略 */

void
iderw(struct buf *b)
{
  struct buf **pp;

  if(!holdingsleep(&b->lock))
    panic("iderw: buf not locked");
  if((b->flags & (B_VALID|B_DIRTY)) == B_VALID)
    panic("iderw: nothing to do");
  if(b->dev != 0 && !havedisk1)
    panic("iderw: ide disk 1 not present");

  acquire(&idelock);  //DOC:acquire-lock

  // Append b to idequeue.
  b->qnext = 0;
  for(pp=&idequeue; *pp; pp=&(*pp)->qnext)  //DOC:insert-queue
    ;
  *pp = b;

  // Start disk if necessary.
  if(idequeue == b)
    idestart(b);

  // Wait for request to finish.
  while((b->flags & (B_VALID|B_DIRTY)) != B_VALID){
    sleep(b, &idelock);
  }


  release(&idelock);
}
```

idestart関数はideにbuf構造体に応じたリクエストを出す。  
ideコントローラのレジスタに書き込みを行うことでそれを行う。

ide.cの1行目に、これはPIOベースのシンプルなIDEドライバコードであるとのコメントがある。

アクセスしているレジスタとデータの意味は[「OSDev ATA PIO Mode」（リンク23）](https://wiki.osdev.org/ATA_PIO_Mode)に記載されている。
関数内に出てくる0x1f7はリンクの「Addressing Modes」の「Registers」によると、読み込み時はステータスレジスタ、書き込み時はコマンドレジスタとなり、コマンドは[「OSDev ATA Command Matrix」（リンク24）](https://wiki.osdev.org/ATA_Command_Matrix)に一覧としてまとまっている。  

読み書きを行うブロックの指定にはLBA方式を使用する。
LBAについては[「OSDev LBA」（リンク25）](https://wiki.osdev.org/LBA)に記載されている。
フロッピーディスクなどの場合はCHSという方法でセクタにアクセスする。

変数sector\_per\_blockからwrite\_cmdまでの値は以下。
- sector\_per\_block: いずれも512バイトなので1
- sector: ブロックがある（はじまる）セクタ番号（LBA）。
ブロックサイズとセクタサイズが等しいので、ブロックサイズがそのまま求める番号となる。
- read\_cmd: 読み込みに使用するコマンド。
コマンドは「OSDev ATA Command Matrix」（リンク24）に載っている。
セクタサイズとブロックサイズが異なる場合、複数のセクタを読む0xc4（IDE\_CMD\_RDMUL）とし、等しい場合はそのセクタだけを読む0x20（IDE\_CMD\_READ）とする。
- write\_cmd: 書き込みに使用するコマンド。
複数セクタを書く場合は0xc5（IDE\_CMD\_WRMUL）、ひとつの場合は0x30（IDE\_CMD\_WRITE）とする。

ideにコマンドをリクエストする前に、ブロックサイズがセクタサイズの7倍より大きくないことを確認するが、なぜこのチェックが必要なのかはわからない。

idewait関数でドライブがビジーでもエラーでもなくなるまで待つ。

読み書きの設定は以下。
デバイスコントロールレジスタ（0x3f6）に0を書き込み、1bit目（nIEN）を0にすることでドライブの割り込みを有効化する。  
セクタカウントレジスタ（0x1f2）に、sector\_per\_blockを設定する。ここでは1。  
LBAloレジスタ（0x1f3）に、LBAの1バイト目を設定する。  
レジスタが8bitなので、LBAは4つに分けて設定する。  
LBAmidレジスタ（0x1f4）に、LBAの2バイト目を設定する。  
LBAhiレジスタ（0x1f5）に、LBAの3バイト目を設定する。  
ドライブ/ヘッドレジスタ（0x1f6）の各bitを設定する。
「OSDev ATA PIO Mode」（リンク23）の「Addressing Modes」の「Drive / Head Register (I/O base + 6)」を見ると各bitの役割が分かる。
5bitと7bitは常に1、6bitはLBAを使用する場合1とするため、ここでは0xe0をセット。
4bitはドライブ番号なのでbuf構造体のdevフィールドの1bitをセット。
0～3bitにはLBAの24～27bitをセットする。

ideにコマンドを発行する。
buf構造体のflagsフィールドの変更済みフラグ（B\_DIRTY）が立っている場合は書き込み、そうでなければ読み込みを行う。  

**書き込みの場合:**  
コマンドレジスタ（0x1f7）に書き込みコマンド（write\_cmd）を書き込み、データレジスタ（0x1f0）に[outsl関数](/chapter_05/05_02_kinit2.md#memset関数)でbuf構造体のdataフィールドの内容を128回に分けて4バイトずつ書き込む。  

**読み込みの場合:**  
コマンドレジスタ（0x1f0）に読み込みコマンド（read\_cmd）を書き込む。  

書き込み終了あるいは、読み込み準備の完了により、ideコントローラから割り込みが入り、最終的にはideintr関数が呼び出される。

ide.c
```c
static void
idestart(struct buf *b)
{
  if(b == 0)
    panic("idestart");
  if(b->blockno >= FSSIZE)
    panic("incorrect blockno");
  int sector_per_block =  BSIZE/SECTOR_SIZE;
  int sector = b->blockno * sector_per_block;
  int read_cmd = (sector_per_block == 1) ? IDE_CMD_READ :  IDE_CMD_RDMUL;
  int write_cmd = (sector_per_block == 1) ? IDE_CMD_WRITE : IDE_CMD_WRMUL;

  if (sector_per_block > 7) panic("idestart");

  idewait(0);
  outb(0x3f6, 0);  // generate interrupt
  outb(0x1f2, sector_per_block);  // number of sectors
  outb(0x1f3, sector & 0xff);
  outb(0x1f4, (sector >> 8) & 0xff);
  outb(0x1f5, (sector >> 16) & 0xff);
  outb(0x1f6, 0xe0 | ((b->dev&1)<<4) | ((sector>>24)&0x0f));
  if(b->flags & B_DIRTY){
    outb(0x1f7, write_cmd);
    outsl(0x1f0, b->data, BSIZE/4);
  } else {
    outb(0x1f7, read_cmd);
  }
}
```


idewait関数はドライブがビジーでもエラーでもなくなるまでwhileループで待つ。  
引数checkerrに0を渡すとエラーチェックはせず、1を渡すとドライブのエラーをチェックし、エラーの場合に-1を返してくれる。  
ループの終了条件はステータスレジスタ（0x1f7）を読み取り、7bitのビジービット（IDE\_BSY）が0、かつ6bitのレディビット（IDE\_DRDY）が1になること。  
エラーチェックではステータスレジスタの5bitドライブ障害エラービット（IDE\_DF）あるいは0bitエラービット（IDE\_ERR）を確認し、いずれかが1の場合に-1を返す。

ide.c
```c
#define IDE_BSY       0x80
#define IDE_DRDY      0x40
#define IDE_DF        0x20
#define IDE_ERR       0x01

/* 略 */

static int
idewait(int checkerr)
{
  int r;

  while(((r = inb(0x1f7)) & (IDE_BSY|IDE_DRDY)) != IDE_DRDY)
    ;
  if(checkerr && (r & (IDE_DF|IDE_ERR)) != 0)
    return -1;
  return 0;
}
```

まだ割り込みベクタの初期化を読むのは先になるが、動きとしては、このあとディスクの準備ができるとideコントローラからirq14番で割り込みが入り、IOAPICのリダイレクトテーブルによって割り込みベクタ 46(32 + 14)番がtrap関数にわたされ、ideintr関数により読み込みが行われる。

ideintr関数はidequeueの先頭のbuf構造体のflagsフィールドに応じてディスクから読み込みを行う。  

もしもbuf構造体のflagsフィールドの変更済みフラグ（B\_DIRTY）が0で、かつidewait関数でドライブの準備ができている場合、ディスクから読み出しを行う。
読み出しは[insl関数](/chapter_03/03_02_bootmain.md#inb関数outb関数insl関数)でコントロールレジスタ（0x1f0）からbuf構造体のdataフィールドにブロックサイズ分読み込む。
次に、buf構造体のflagsフィールドの読み込み済みフラグ（B\_VALID）を1にし、変更済みフラグ（B\_DIRTY）をビット反転して論理積を取って0にする。

読み込みを待っているプロセスを起床し、idequeueにエントリがまだ残っている場合は次のエントリを引数としてidestart関数を呼び出す。

ide.c
```c
void
ideintr(void)
{
  struct buf *b;

  // First queued buffer is the active request.
  acquire(&idelock);

  if((b = idequeue) == 0){
    release(&idelock);
    return;
  }
  idequeue = b->qnext;

  // Read data if needed.
  if(!(b->flags & B_DIRTY) && idewait(1) >= 0)
    insl(0x1f0, b->data, BSIZE/4);

  // Wake process waiting for this buf.
  b->flags |= B_VALID;
  b->flags &= ~B_DIRTY;
  wakeup(b);

  // Start disk on next buf in queue.
  if(idequeue != 0)
    idestart(idequeue);

  release(&idelock);
}
```


## データブロックの割り当て（balloc関数）
balloc関数はファイルシステムのdata領域で空いているブロックを探してそのブロック番号を返す。
ファイルシステムのbitmap領域に当該ブロックを使用済みとしてマークする。

bitmap領域に関しては、教科書[「xv6 a simple, Unix-like teaching operating system」（リンク1）](https://pdos.csail.mit.edu/6.828/2018/xv6/book-rev11.pdf)の6章「File system」の「Code: Block allocator」で説明されている。  
bitmap領域の1bitがファイルシステム上の1ブロックを表しており、使用済みか否かを管理している。
bitmap領域が何番目のブロックから始まるのかといったファイルシステムの情報は、superblockにある。
これは後々initプロセス実行時に呼び出されるforkret関数の中でiinit関数を呼び出し、superblock構造体の大域変数sbに読み込む。

この関数は二重のforループになっており、外側でbitmap領域のブロックを、内側でそのブロックのbitを走査する。  
**外側ループ:**  
変数bはbitmap領域のbitの番号を表すと同時に、ファイルシステムのブロック番号でもある。
BPBマクロはブロックあたりのbitmapのbit数を表す（512\*8）ので、bはブロックに含まれるbit数（4096）ずつ増加することになる。
終了条件はbがファイルシステムの総ブロックサイズsb.size（1000ブロック）を越えるときなので、fs.imgの場合はこのループは1度しか回らない。  
次にbitmap領域のブロックをbuf構造体のポインタbpに取り出す。
ブロックの取り出しにはbread関数を使用し、ブロック番号の指定はBBLOCKマクロで行う。
BBLOCKマクロはbitmap領域のbビット目が含まれるブロック番号を返してくれる。
これはbをBPBで割ってブロック番号を割り出し、bitmap領域の開始ブロックsb.bmapstartに加算して求めている。

**内側ループ:**  
bitmap領域の現在のブロックのbitを1つずつ走査し、0になっているbit（未使用のブロック番号）を探す。  
変数biをブロックの総bit数（512\*8）までインクリメントしていく。
ブロックの全てのbitを走査すると言ってもbuf構造体bpのdataフィールドからアクセスできるのは1バイトずつなので、変数mを8bit分のフラグとして、0bitから順に7bitまで1を立てていってその論理積でbitを調べる。
なので、mにはbiを8で割った余りだけ左シフトした1を代入していく。
bpのdataフィールドの各bitをmで論理積してブロックの使用状況を調べる。
もしもbitが0で、ブロックb+biが使用されていないとき、今度はmの論理和でbpのdataフィールドのbitを立てて使用済みにマークする。
bpが更新されたので、log\_write関数で変更をディスクに反映し、brelse関数でbpを解放する。
bzero関数で使用済みにマークしたブロック（b+bi番目）を0埋めし、呼び出し元にブロック番号を返す。
bzero関数は引数devのデバイスの引数bnoのブロック番号で示されるブロックを0埋めして、log\_write関数でディスクに書き出す。  
もしもbitmap領域のbitが全て1で、未使用のブロックが存在しないときは、panicする。

fs.h
```c
#define BPB           (BSIZE*8)

/* 略 */

#define BBLOCK(b, sb) (b/BPB + sb.bmapstart)
```

fs.c
```c
static void
bzero(int dev, int bno)
{
  struct buf *bp;

  bp = bread(dev, bno);
  memset(bp->data, 0, BSIZE);
  log_write(bp);
  brelse(bp);
}

/* 略 */

static uint
balloc(uint dev)
{
  int b, bi, m;
  struct buf *bp;

  bp = 0;
  for(b = 0; b < sb.size; b += BPB){
    bp = bread(dev, BBLOCK(b, sb));
    for(bi = 0; bi < BPB && b + bi < sb.size; bi++){
      m = 1 << (bi % 8);
      if((bp->data[bi/8] & m) == 0){  // Is block free?
        bp->data[bi/8] |= m;  // Mark block in use.
        log_write(bp);
        brelse(bp);
        bzero(dev, b + bi);
        return b + bi;
      }
    }
    brelse(bp);
  }
  panic("balloc: out of blocks");
}
```


## データブロックの解放（bfree関数）
この関数はデバイスdevのb番目のブロックを解放する。  
balloc関数と同じ要領でファイルシステムのbitmap領域のb番目のbitを0にする。  
log\_write関数でディスクに変更を反映し、brelse関数でバッファキャッシュを解放する。  

fs.c
```c
static void
bfree(int dev, uint b)
{
  struct buf *bp;
  int bi, m;

  bp = bread(dev, BBLOCK(b, sb));
  bi = b % BPB;
  m = 1 << (bi % 8);
  if((bp->data[bi/8] & m) == 0)
    panic("freeing free block");
  bp->data[bi/8] &= ~m;
  log_write(bp);
  brelse(bp);
}
```

