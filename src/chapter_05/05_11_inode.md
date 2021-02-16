# 5.11. iノード
スケジューラの実行までに使用するiノード周りの操作について書く。

- [inode構造体](#inode構造体)
- [inode構造体の作成（iget関数）](#inode構造体の作成iget関数)
- [ロック（ilock/iunlock関数）](#ロックilockiunlock関数)
- [参照カウンタのインクリメント（idup数）](#参照カウンタのインクリメントidup関数)
- [参照カウンタのデクリメント（iput関数）](#参照カウンタのデクリメントiput関数)
- [ファイルデータの取得（readi関数）](#ファイルデータの取得readi関数)
- [ブロックの割り当て（bmap関数）](#ブロックの割り当てbmap関数)
- [削除（itrunc関数）](#削除itrunc関数)
- [更新（iupdate関数）](#更新iupdate関数)


## inode構造体
inodeについては、[「xv6 a simple, Unix-like teaching operating system」（リンク1）](https://pdos.csail.mit.edu/6.828/2018/xv6/book-rev11.pdf)の6章「File system」を読むとよい。

dinode構造体とinode構造体の2つがある。  

**dinode構造体**  
ディスク上のiノードブロックにその情報が格納されてる。
typeフィールドは次の4つの状態を表す。
- 0: 未使用
- 1: ファイル
- 2: ディレクトリ
- 3: デバイスファイル

addrs配列はファイルの存在しているディスク上のブロック番号を持ち、「xv6 a simple, Unix-like teaching operating system」（リンク1）の図6-3「The representation of a file on disk」で分かりやすく図解されている。
要素数は13で、12番目（NDIRECT）までは直接ブロック番号が入っており、13番目はindirect blockのブロック番号が入っている。
ブロックサイズは512バイトで、ブロック番号は4バイトなので、indirect blockは128個のブロックを指すことができる。
つまりinode構造体は全部で140個（128+12）のブロックを指すことができる。
逆に言えば71680バイト（140\*512）がファイルの最大サイズとなる。


にあるように、前12エントリがそのままデータブロックを指しており、13エントリ目は128エントリまで保持できる別のテーブルを参照している。
ブロックサイズが512バイトなので、dinode構造体から直接参照できるのは`12 * 512 = 6kB`まで。

fs.h
```c
#define NDIRECT 12

/* 略 */

struct dinode {
  short type;           // File type
  short major;          // Major device number (T_DEV only)
  short minor;          // Minor device number (T_DEV only)
  short nlink;          // Number of links to inode in file system
  uint size;            // Size of file (bytes)
  uint addrs[NDIRECT+1];   // Data block addresses
};
```

**inode構造体**  
後半はdinode構造体のコピーになっている。  
validフィールドは既にディスクからdinodeの情報を読み出したか否かを示すフラグ。

file.h
```c
struct inode {
  uint dev;           // Device number
  uint inum;          // Inode number
  int ref;            // Reference count
  struct sleeplock lock; // protects everything below here
  int valid;          // inode has been read from disk?

  short type;         // copy of disk inode
  short major;
  short minor;
  short nlink;
  uint size;
  uint addrs[NDIRECT+1];
};
```
inode構造体はicache構造体に持っておく。  
全部で50個持てる。逆に言えば50個以上開けない。

param.h
```c
#define NINODE       50  // maximum number of active i-nodes
```

fs.c
```c
struct {
  struct spinlock lock;
  struct inode inode[NINODE];
} icache;
```


## inode構造体の作成（iget関数）

iノードキャッシュからinode構造体を返す。  
探しているiノードがキャッシュにある場合は参照カウンタをインクリメントしてそれを返し、無い場合はキャッシュ内の未使用エントリを初期化して返す。
ディスクからのデータ読み出しは行わないので、dinode構造体の持つ内容のコピーは行わない。
キャッシュからはデバイス番号とiノード番号を頼りにエントリを検索する。  

fs.c
```c
static struct inode*
iget(uint dev, uint inum)
{
  struct inode *ip, *empty;

  acquire(&icache.lock);

  // Is the inode already cached?
  empty = 0;
  for(ip = &icache.inode[0]; ip < &icache.inode[NINODE]; ip++){
    if(ip->ref > 0 && ip->dev == dev && ip->inum == inum){
      ip->ref++;
      release(&icache.lock);
      return ip;
    }
    if(empty == 0 && ip->ref == 0)    // Remember empty slot.
      empty = ip;
  }

  // Recycle an inode cache entry.
  if(empty == 0)
    panic("iget: no inodes");

  ip = empty;
  ip->dev = dev;
  ip->inum = inum;
  ip->ref = 1;
  ip->valid = 0;
  release(&icache.lock);

  return ip;
}
```


## ロック（ilock/iunlock関数）
iノードを使用する処理はファイルシステムを扱う処理なのでディスクにアクセスする可能性があり、長時間ロックを取得する可能性が高いためスリープロックを使用する。

ilock関数はinode構造体のロックを取り、dinode構造体のデータをまだ持っていない場合にバッファキャッシュやディスクからそれを読み込む。  

dinode構造体のデータはbread関数で取得する。
bread関数はデータをデバイス番号とブロック番号を基にバッファキャッシュから探し、そこに無ければディスクからデータを読み込んでバッファキャッシュに加える。
ブロック番号はIBLOCKマクロを使用して、iノード番号とsuperblock構造体から求める。
superblock構造体は、大域変数sbがiinit関数で定義される。
スーパーブロックは[「xv6 a simple, Unix-like teaching operating system」（リンク1）](https://pdos.csail.mit.edu/6.828/2018/xv6/book-rev11.pdf)の図6-2における1番目のブロックで、ファイルシステムのサイズやデータブロック数等の情報を持っている。
IBLOCKマクロはinode番号をブロック当たりのiノード数で割り、それにiノード領域の開始ブロック番号を加算することで目的のiノードが含まれるブロック番号を求める。
ブロック当たりのiノード数はIPBとして定義されており、ブロックサイズをdinode構造体のサイズで割ったもの。

buf構造体のdataフィールドにはdinode構造体のデータがiノード番号順に並んでいるため、iノード番号をIPBで割った余り番目からデータを取り出す。  
その後dinode構造体の全フィールドをinode構造体にコピーし、最後にバッファキャッシュを始末する。

fs.h
```c
#define IPB           (BSIZE / sizeof(struct dinode))

// Block containing inode i
#define IBLOCK(i, sb)     ((i) / IPB + sb.inodestart)
```

fs.c
```c
struct superblock sb;

/* 略 */

void
ilock(struct inode *ip)
{
  struct buf *bp;
  struct dinode *dip;

  if(ip == 0 || ip->ref < 1)
    panic("ilock");

  acquiresleep(&ip->lock);

  if(ip->valid == 0){
    bp = bread(ip->dev, IBLOCK(ip->inum, sb));
    dip = (struct dinode*)bp->data + ip->inum%IPB;
    ip->type = dip->type;
    ip->major = dip->major;
    ip->minor = dip->minor;
    ip->nlink = dip->nlink;
    ip->size = dip->size;
    memmove(ip->addrs, dip->addrs, sizeof(ip->addrs));
    brelse(bp);
    ip->valid = 1;
    if(ip->type == 0)
      panic("ilock: no type");
  }
}
```

ロックはiunlock関数で解放する。  

fs.c
```c
void
iunlock(struct inode *ip)
{
  if(ip == 0 || !holdingsleep(&ip->lock) || ip->ref < 1)
    panic("iunlock");

  releasesleep(&ip->lock);
}
```


## 参照カウンタのインクリメント（idup関数）
inode構造体の参照カウンタをインクリメントする。

fs.c
```c
struct inode*
idup(struct inode *ip)
{
  acquire(&icache.lock);
  ip->ref++;
  release(&icache.lock);
  return ip;
}
```


## 参照カウンタのデクリメント（iput関数）
この関数は引数のinode構造体ipの参照カウンタをデクリメントする。
また、ファイルデータをディスクから取得済みかつ他のファイル（ディレクトリエントリ）がリンクされておらず、誰からも参照されていない場合にiノードを解放する。
ここで言う解放は、addrsをバッファキャッシュごと全て解放しディスクに書き出すことでファイルを削除することを指す。
参照カウンタの値は誰からも参照されていない場合は1。

iunlockput関数はinode構造体のロックを解放し、iput関数を呼び出す。

fs.c
```c
void
iput(struct inode *ip)
{
  acquiresleep(&ip->lock);
  if(ip->valid && ip->nlink == 0){
    acquire(&icache.lock);
    int r = ip->ref;
    release(&icache.lock);
    if(r == 1){
      // inode has no links and no other references: truncate and free.
      itrunc(ip);
      ip->type = 0;
      iupdate(ip);
      ip->valid = 0;
    }
  }
  releasesleep(&ip->lock);

  acquire(&icache.lock);
  ip->ref--;
  release(&icache.lock);
}

/* 略 */

void
iunlockput(struct inode *ip)
{
  iunlock(ip);
  iput(ip);
}
```


## ファイルデータの取得（readi関数）
この関数はファイルからデータを最大nバイト分dstに読み込む。

iノードがデバイスファイルの場合とそれ以外（ファイルあるいはディレクトリ）とで処理が分かれる。

**デバイスファイルの場合:**  
読み込みにはデバイス番号を頼りに[devsw配列](https://kkmtyyz.github.io/xv6-notebook/chapter_05/05_09_consoleinit.html)の該当するread関数を使う。  
例えばコンソールならdevsw[1].readなので、consoleinit関数で設定した[consoleread関数](https://kkmtyyz.github.io/xv6-notebook/chapter_05/05_09_consoleinit.html#consoleread関数)が実行される。

**ファイルあるいはディレクトリの場合:**  
ファイルのデータをoffバイト目から読み込み始める。  
ファイルサイズよりも読み込みサイズ（off+n）大きい場合は調整する。  
読み込むデータ（nバイト）がブロックを跨いでいる可能性があるため、ループで読み込む。  
mがループ毎にdstにコピーするバイト数を示し、totにはコピーした総バイト数を持つ。  
読み込みにはbread関数を使う。
この関数にはデバイス番号とブロック番号を渡す必要がある。
[bmap関数](#ブロックの割り当てbmap関数)では第一引数のinode構造体のaddrsフィールドから、第二引数で指定されたバイト目があるブロック番号を得ることができる。  
基本的にはoffからそのブロックの終わりまでずつコピーしていく。  
例えばoffが600の場合、1ブロック512バイトなので、offは2ブロック目の88バイト目から始まる。
なので `512-88=424` バイトずつコピーしていく。
そして読み込みバイト数nがコピーしていく単位で割り切れない場合は、最後のループでnからコピー済みバイト数totの差だけコピーする。  
他の例として、nが1000の場合、3回目のループで残りコピーバイト数が `1000-424*2=152` なので、152バイトだけコピーする。
変数mにはこの動きをするために、minマクロを使用して残りコピーバイト数（n-tot）と基本的なコピー単位のどちらか小さい方を代入する。  
コピーバイト数mが決まると、[memmove関数](https://kkmtyyz.github.io/xv6-notebook/chapter_05/05_09_consoleinit.html#memmove関数)でdstにブロックのデータ（bs-\>data）のオフセットの位置からその分だけコピーする。  
バッファキャッシュbpは読み込み後不要となるのでループ毎に解放する。
1ブロック分ずつコピーするわけではないので、次のループでも同じブロックをバッファキャッシュに読み込む可能性がある。
しかしここで解放しなければ、バッファキャッシュのサイズが30なのでコピー対象のデータが30ブロック以上の時にキャッシュが足りなくなりpanicしてしまう。

fs.c
```c
#define min(a, b) ((a) < (b) ? (a) : (b))

/* 略 */

int
readi(struct inode *ip, char *dst, uint off, uint n)
{
  uint tot, m;
  struct buf *bp;

  if(ip->type == T_DEV){
    if(ip->major < 0 || ip->major >= NDEV || !devsw[ip->major].read)
      return -1;
    return devsw[ip->major].read(ip, dst, n);
  }

  if(off > ip->size || off + n < off)
    return -1;
  if(off + n > ip->size)
    n = ip->size - off;

  for(tot=0; tot<n; tot+=m, off+=m, dst+=m){
    bp = bread(ip->dev, bmap(ip, off/BSIZE));
    m = min(n - tot, BSIZE - off%BSIZE);
    memmove(dst, bp->data + off%BSIZE, m);
    brelse(bp);
  }
  return n;
}
```


## ブロックの割り当て（bmap関数）
この関数はaddrsフィールドのbn番目のブロック番号を返す。
もしもbn番目が0でブロックが割り当てられていない場合、ディスクのdata領域から1ブロック割り当てる。

2つのif文に分かれており、前半はbnがaddrsの12番目までの処理を行い、後半はindirect block（13番目）の処理を行う。

**12番目まで:**  
addrsフィールドのbn番目からブロック番号を取り出す。
もしもbn番目が0でブロックが割り当てられていない場合、balloc関数でファイルシステムのdata領域から1ブロック割り当て、そのブロック番号をbn番目に入れる。

**indirect block:**  
bnをそのままindirect block内でのインデックスとするため、12減算する。  
NINDIRECTはindirect blockのエントリ数。  
indirect blockのブロック番号を取り出し、0の場合はballoc関数で1ブロック割り当てる。  
変数bpにbread関数でindirect blockを読み出し、そこからさらにbn番目のブロック番号を取り出す。
bn番目が0の場合も同様にブロックを割り当てる。
このとき、新たにブロックの割り当てを行うとindirect blockの内容を更新することになるため、log\_write関数で変更をディスクに反映する。
log\_write関数ではバッファキャッシュのB\_DIRTYを立て、後々呼び出されるwrite\_log関数でディスクに書き込みを行う。  
bpをbrelse関数で解放し、呼び出し元にブロック番号addrを返して終了。

fs.h
```c
#define NDIRECT 12
#define NINDIRECT (BSIZE / sizeof(uint))
```

fs.c
```c
static uint
bmap(struct inode *ip, uint bn)
{
  uint addr, *a;
  struct buf *bp;

  if(bn < NDIRECT){
    if((addr = ip->addrs[bn]) == 0)
      ip->addrs[bn] = addr = balloc(ip->dev);
    return addr;
  }
  bn -= NDIRECT;

  if(bn < NINDIRECT){
    // Load indirect block, allocating if necessary.
    if((addr = ip->addrs[NDIRECT]) == 0)
      ip->addrs[NDIRECT] = addr = balloc(ip->dev);
    bp = bread(ip->dev, addr);
    a = (uint*)bp->data;
    if((addr = a[bn]) == 0){
      a[bn] = addr = balloc(ip->dev);
      log_write(bp);
    }
    brelse(bp);
    return addr;
  }

  panic("bmap: out of range");
}
```


## 削除（itrunc関数）
この関数はiノードのデータを解放する。
ここで言う解放は、addrsをバッファキャッシュごと全て解放しディスクに書き出すことでファイルを削除することを指す。

ファイルシステム上のブロックを未使用にマークする。  
動作はbmap関数とほぼ同様で、addrsフィールドを走査してbfree関数でブロックを解放しつつ0を代入していく。  
その後ファイルサイズも0とし、iupdate関数を呼び出してdinode構造体に変更を反映する。

fs.c
```c
static void
itrunc(struct inode *ip)
{
  int i, j;
  struct buf *bp;
  uint *a;

  for(i = 0; i < NDIRECT; i++){
    if(ip->addrs[i]){
      bfree(ip->dev, ip->addrs[i]);
      ip->addrs[i] = 0;
    }
  }

  if(ip->addrs[NDIRECT]){
    bp = bread(ip->dev, ip->addrs[NDIRECT]);
    a = (uint*)bp->data;
    for(j = 0; j < NINDIRECT; j++){
      if(a[j])
        bfree(ip->dev, a[j]);
    }
    brelse(bp);
    bfree(ip->dev, ip->addrs[NDIRECT]);
    ip->addrs[NDIRECT] = 0;
  }

  ip->size = 0;
  iupdate(ip);
}
```


## 更新（iupdate関数）
この関数はinode構造体の内容をdinode構造体にコピーし、その変更をディスクに反映する。

ディスク上にあるdinode構造体はbread関数で取得する。
その際のブロック番号は[IBLOCKマクロ](#ロックilockiunlock関数)で算出する。  
log\_write関数で変更をディスクに反映する。
この関数ではバッファキャッシュのB\_DIRTYを立て、後々呼び出されるwrite\_log関数でディスクに書き込みを行う。  

fs.c
```c
void
iupdate(struct inode *ip)
{
  struct buf *bp;
  struct dinode *dip;

  bp = bread(ip->dev, IBLOCK(ip->inum, sb));
  dip = (struct dinode*)bp->data + ip->inum%IPB;
  dip->type = ip->type;
  dip->major = ip->major;
  dip->minor = ip->minor;
  dip->nlink = ip->nlink;
  dip->size = ip->size;
  memmove(dip->addrs, ip->addrs, sizeof(ip->addrs));
  log_write(bp);
  brelse(bp);
}
```
