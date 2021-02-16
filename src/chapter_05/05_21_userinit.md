# 5.21. userinit関数
initcode.Sのプロセスを作成し、プロセステーブルに追加する。

userinit関数を見る前に、initプロセスを実行するinitcode.Sと、プロセステーブルのエントリをプロセスとして割り当てるallocproc関数を見る。

## initcode.S
[entryother.S](https://kkmtyyz.github.io/xv6-notebook/chapter_05/05_19_startothers.html)と同様。
TEXTセグメントの開始アドレスは0x0。
以下のシンボルが作成され、バイナリがカーネルにリンクされる。

- \_binary\_initcode\_start
- \_binary\_initcode\_end
- \_binary\_initcode\_size

Makefile
```makefile
initcode: initcode.S
  $(CC) $(CFLAGS) -nostdinc -I. -c initcode.S
  $(LD) $(LDFLAGS) -N -e start -Ttext 0 -o initcode.out initcode.o
  $(OBJCOPY) -S -O binary initcode.out initcode
  $(OBJDUMP) -S initcode.o > initcode.asm
```

## allocproc関数
この関数はプロセステーブルから未使用のエントリを取得し、初期化して返す。

[プロセステーブル](https://kkmtyyz.github.io/xv6-notebook/chapter_05/05_14_pinit.html)を走査し、stateフィールドがUNUSEDのエントリを取得してfoundラベルにジャンプする。

foundラベル以降ではproc構造体の各フィールドと、カーネルスタックを初期化する。  
[kalloc関数](https://kkmtyyz.github.io/xv6-notebook/chapter_05/05_03_kvmalloc.html#kalloc関数)で1ページ分のをカーネルスタックとして割り当てる。
カーネルスタックの底（p-\>state + KSTACKSIZE）から順に以下3つを設定する。
- プロセスの状態を保存するために使用するトラップフレームの領域を確保する。
- 4バイト分の領域を確保し、trapret関数のアドレスを代入する。
カーネル空間での処理終了後、trapret関数を呼び出してトラップフレームに保存した状態を復元し、最終的にiret命令でユーザ空間に戻る。
- contextフィール分の領域を確保し、0埋めする。
context構造体のeipフィールドにforkret関数のアドレスを代入する。

プロセスは最初のコンテキストスイッチ時にforkret関数の実行から始まり、次にtrapret関数が実行され、トラップフレームに保存した状態を復元してユーザ空間でプログラムを実行することになる。

forkret関数とtrapret関数についてはinitプロセスで見る。

proc.h
```c
enum procstate { UNUSED, EMBRYO, SLEEPING, RUNNABLE, RUNNING, ZOMBIE };
```

proc.c
```c
static struct proc*
allocproc(void)
{
  struct proc *p;
  char *sp;

  acquire(&ptable.lock);

  for(p = ptable.proc; p < &ptable.proc[NPROC]; p++)
    if(p->state == UNUSED)
      goto found;

  release(&ptable.lock);
  return 0;

found:
  p->state = EMBRYO;
  p->pid = nextpid++;

  release(&ptable.lock);

  // Allocate kernel stack.
  if((p->kstack = kalloc()) == 0){
    p->state = UNUSED;
    return 0;
  }
  sp = p->kstack + KSTACKSIZE;

  // Leave room for trap frame.
  sp -= sizeof *p->tf;
  p->tf = (struct trapframe*)sp;

  // Set up new context to start executing at forkret,
  // which returns to trapret.
  sp -= 4;
  *(uint*)sp = (uint)trapret;

  sp -= sizeof *p->context;
  p->context = (struct context*)sp;
  memset(p->context, 0, sizeof *p->context);
  p->context->eip = (uint)forkret;

  return p;
}
```


## userinit関数
変数pに、allocproc関数を使用してプロセステーブルからエントリを割り当てる。
このプロセスは最初initcode.Sを実行するために使用されるが、後からexecシステムコールによりinit.cを実行するプロセスに変わる。
そのため、initプロセスを持つためのproc.cの変数initprocに代入しておく。  
プロセスのpgdirフィールドにカーネル空間のページディレクトリエントリを持ったページディレクトリを作成する。
作成は[setupkvm関数](https://kkmtyyz.github.io/xv6-notebook/chapter_05/05_03_kvmalloc.html#setupkvm関数)でvm.cのkmap配列を基に行われる。
inituvm関数で、initcode.Sを配置するためのページを割り当て、ページディレクトリにPDEとPTEを作成する。
initcode.SのTEXTセクションは仮想アドレス0から始まるので、0番目のPDEから変換するように作る。  

initプロセスのトラップフレームの設定を行う。  
スケジューラによりinitプロセスにコンテキストスイッチされたとき、initプロセスのcontextフィールドのeipの値により、forkret関数が実行される。
さらにforkret関数からiret命令によりtrapret関数が実行され、トラップフレームの内容にしたがってプログラムが実行される。
ここではその準備をする。
- **cs:** [ユーザコードセグメントディスクリプタ](https://kkmtyyz.github.io/xv6-notebook/chapter_05/05_06_seginit.html)のエントリ番号（3）と、DPL3を設定する。
- **ds:** 同様にユーザデータセグメントディスクリプタのエントリ番号（4）と、DPL3を設定する。
- **es, ss:** データセグメントディスクリプタを設定する。
特に使用しないので、とりあえずデータセグメントディスクリプタを設定しているんだと思う。
- **eflags:** 9bit（Interrupt Enable Flag）のみを1に設定する（0x00000200）。
つまりユーザ空間に戻った後は割り込みが有効化される。
eflagsレジスタの各bitの役割については[「Intel 64 and IA-32 architectures software developer's manual combined volumes: 1, 2A, 2B, 2C, 2D, 3A, 3B, 3C, 3D, and 4」（リンク8）](https://software.intel.com/content/www/us/en/develop/download/intel-64-and-ia-32-architectures-sdm-combined-volumes-1-2a-2b-2c-2d-3a-3b-3c-3d-and-4.html)のVol.1「3.4.3 EFLAGS Register」の図3-8 「EFLAGS Register」に記載されている。
- **esp:** ページサイズ（4096）を設定する。
initcode.Sには1ページ分の領域しか割り当てないので、その中で最も高いアドレスにスタックポインタを設定する。
- **eip:** initcode.Sは仮想アドレス0から開始するようにリンクされているため0を設定する。

プロセスの名前に「initcode」を設定する。
[safestrcpy関数](#safestrcpy関数)で即値 "initcode" をコピーしている。  
cwdフィールドにルートディレクトリのinode構造体を設定する。
namei関数はinode構造体を取得する。  
プロセスを実行する準備が整ったので、状態を実行可能状態とする。  
ここで、コメントにもあるように、他のAPでは既にスケジューラが動いているため、initプロセスが必ずしもBSPで実行されるとは限らない。
MakefileのCPUSを8とかにしてGDBでscheduler関数の途中で止めてinitcodeプロセスが実行されるときのapicidを確認すると、APで実行されるパターンを観測できる。

proc.c
```c
void
userinit(void)
{
  struct proc *p;
  extern char _binary_initcode_start[], _binary_initcode_size[];

  p = allocproc();
  
  initproc = p;
  if((p->pgdir = setupkvm()) == 0)
    panic("userinit: out of memory?");
  inituvm(p->pgdir, _binary_initcode_start, (int)_binary_initcode_size);
  p->sz = PGSIZE;
  memset(p->tf, 0, sizeof(*p->tf));
  p->tf->cs = (SEG_UCODE << 3) | DPL_USER;
  p->tf->ds = (SEG_UDATA << 3) | DPL_USER;
  p->tf->es = p->tf->ds;
  p->tf->ss = p->tf->ds;
  p->tf->eflags = FL_IF;
  p->tf->esp = PGSIZE;
  p->tf->eip = 0;  // beginning of initcode.S

  safestrcpy(p->name, "initcode", sizeof(p->name));
  p->cwd = namei("/");

  // this assignment to p->state lets other cores
  // run this process. the acquire forces the above
  // writes to be visible, and the lock is also needed
  // because the assignment might not be atomic.
  acquire(&ptable.lock);

  p->state = RUNNABLE;

  release(&ptable.lock);
}
```

## inituvm関数
この関数はinitcode.Sをinitプロセスの0番目のPDEに配置するためだけに存在している。

pgdirにinitプロセスのページディレクトリのアドレスを受け、initにinitcode.Sの開始アドレス、szにinitcode.Sのサイズを受ける。  
まずszがページサイズ以上だった場合にpanicする。
理由は定かではないけど、恐らく、ページングが有効になっている都合上、initcode.Sが複数ページにわたるとkallocでも複数ページ確保しなければならず、initcode.Sを適切に分割して各ページに配置することが手間になるからだと思う。  
pgdirに仮想アドレス0から1ページ分の領域を、memの物理アドレスを参照するようにPDEとPTEを作成する。
作成には[mappages関数](https://kkmtyyz.github.io/xv6-notebook/chapter_05/05_03_kvmalloc.html#mappages関数)を使用し、PTEの属性は書き込みフラグとユーザフラグを立てる。  
最後にinitcode.Sをmemmove関数でmemにコピーする。
memは[kalloc関数](https://kkmtyyz.github.io/xv6-notebook/chapter_05/05_03_kvmalloc.html#kalloc関数)で割り当てる1ページ分の領域しか持っていないため、initプロセスは4kBに収まる必要がある。

vm.c
```c
void
inituvm(pde_t *pgdir, char *init, uint sz)
{
  char *mem;

  if(sz >= PGSIZE)
    panic("inituvm: more than a page");
  mem = kalloc();
  memset(mem, 0, PGSIZE);
  mappages(pgdir, 0, PGSIZE, V2P(mem), PTE_W|PTE_U);
  memmove(mem, init, sz);
}
```

## safestrcpy関数
この関数はsにtからn-1バイト分だけコピーする。
コピー先sはヌル終端される。

string.c
```c
char*
safestrcpy(char *s, const char *t, int n)
{
  char *os;

  os = s;
  if(n <= 0)
    return os;
  while(--n > 0 && (*s++ = *t++) != 0)
    ;
  *s = 0;
  return os;
}
```

## namei関数
この関数はpathとして受け取ったパスのinode構造体を呼び出し元に返す。
中身としてはnamex関数を呼び出してその戻り値を返すだけ。
namex関数を呼び出す際にはnameiparentフラグ（第二引数）を0にしている。

fs.c
```c
struct inode*
namei(char *path)
{
  char name[DIRSIZ];
  return namex(path, 0, name);
}
```

namex関数は引数pathで指定されたファイルのinode構造体を返す。
また、引数nameにファイル名を代入してくれる。  
引数nameiparentが0以外の場合は、pathの親ファイルのinode構造体を返す。  

変数ipに、ルートディレクトリのinode構造体か、現在のプロセスのカレントディレクトリのinode構造体を代入する。
[iget](https://kkmtyyz.github.io/xv6-notebook/chapter_05/05_11_inode.html#inode構造体の作成iget関数)と[idup](https://kkmtyyz.github.io/xv6-notebook/chapter_05/05_11_inode.html#参照カウンタのインクリメントidup関数)はiノードのところで見た。

whileループでpathのiノードを順に辿り、目的のiノードを探す。
条件式でskipelem関数を実行し、pathのファイル名をひとつずつ消化して、nameにその時のファイル名を代入し、pathを残りのパスで更新する。
つまり、namex関数の引数pathが単一のファイル名だった場合、whileループは行われない。

ループ内の処理を見る。  
ipがディレクトリ以外の場合、呼び出し元に0を返す。
このループに入る時点でipがディレクトリであることが確定しているため、この分岐は実質的なエラー処理にあたる。  
引数nameiparentが真かつpathの最後のループの場合、呼び出し元にnameのディレクトリを返す。
ここではip更新前なのでipはnameのひとつ上のディレクトリのinode構造体になっている。  
ipを更新するため、変数nextにipのディレクトリエントリでファイル名がnameと等しいエントリのinode構造体を取得する。
取得にはdirlookup関数を使用する。
この関数は第一引数のinode構造体のデータをバッファキャッシュから取得し、ディレクトリエントリから第二引数のnameとファイル名が等しいエントリのinodeを返してくれる。
第三引数は目的のディレクトリエントリが何番目だったかを返してくれるが、使用しないのでここでは0。  

whileループが終わったとき、nameには引数pathの最後のファイル名が入っており、そのinode構造体がipに入っている。
引数nameiparentが真のときにwhileループ終了後まで来る場合は不正なので、[iput関数](https://kkmtyyz.github.io/xv6-notebook/chapter_05/05_11_inode.html#参照カウンタのデクリメントiput関数)でipの参照カウンタをデクリメントし、必要であれば解放する。
pathが単一のファイル名だけだった場合がこれにあたる。

fs.c
```c
static struct inode*
namex(char *path, int nameiparent, char *name)
{
  struct inode *ip, *next;

  if(*path == '/')
    ip = iget(ROOTDEV, ROOTINO);
  else
    ip = idup(myproc()->cwd);

  while((path = skipelem(path, name)) != 0){
    ilock(ip);
    if(ip->type != T_DIR){
      iunlockput(ip);
      return 0;
    }
    if(nameiparent && *path == '\0'){
      // Stop one level early.
      iunlock(ip);
      return ip;
    }
    if((next = dirlookup(ip, name, 0)) == 0){
      iunlockput(ip);
      return 0;
    }
    iunlockput(ip);
    ip = next;
  }
  if(nameiparent){
    iput(ip);
    return 0;
  }
  return ip;
}
```

## skipelem関数
この関数は引数pathの最初のファイル名を引数nameにコピーし、残りを戻り値として返す。  
コメントにExampleとして具体的な入出力が記載されている。  
ファイル名は最大14（DIRSIZ）バイト。

fs.c
```c
// Examples:
//   skipelem("a/bb/c", name) = "bb/c", setting name = "a"
//   skipelem("///a//bb", name) = "bb", setting name = "a"
//   skipelem("a", name) = "", setting name = "a"
//   skipelem("", name) = skipelem("////", name) = 0
//
static char*
skipelem(char *path, char *name)
{
  char *s;
  int len;

  while(*path == '/')
    path++;
  if(*path == 0)
    return 0;
  s = path;
  while(*path != '/' && *path != 0)
    path++;
  len = path - s;
  if(len >= DIRSIZ)
    memmove(name, s, DIRSIZ);
  else {
    memmove(name, s, len);
    name[len] = 0;
  }
  while(*path == '/')
    path++;
  return path;
}
```

## dirlookup関数
この関数はディレクトリdpから引数nameと同名のディレクトリエントリを返す。
また、引数poffにディレクトリエントリが何番目だったかを返してくれる。

forループでディレクトリエントリを走査する。
dirent構造体の取得は[readi](https://kkmtyyz.github.io/xv6-notebook/chapter_05/05_11_inode.html#ファイルデータの取得read関数)で行う。

fs.h
```c
#define DIRSIZ 14

struct dirent {
  ushort inum;
  char name[DIRSIZ];
};
```

fs.c
```c
struct inode*
dirlookup(struct inode *dp, char *name, uint *poff)
{
  uint off, inum;
  struct dirent de;

  if(dp->type != T_DIR)
    panic("dirlookup not DIR");

  for(off = 0; off < dp->size; off += sizeof(de)){
    if(readi(dp, (char*)&de, off, sizeof(de)) != sizeof(de))
      panic("dirlookup read");
    if(de.inum == 0)
      continue;
    if(namecmp(name, de.name) == 0){
      // entry matches path element
      if(poff)
        *poff = off;
      inum = de.inum;
      return iget(dp->dev, inum);
    }
  }

  return 0;
}
```

namecmp関数はstrncmp関数で14文字比較する。  
strncmp関数はpとqを最大nバイト比較し、等しければ0、等しくなければ等しくなくなった文字の差を返す。

fs.c
```c
int
namecmp(const char *s, const char *t)
{
  return strncmp(s, t, DIRSIZ);
}
```

string.c
```c
int
strncmp(const char *p, const char *q, uint n)
{
  while(n > 0 && *p && *p == *q)
    n--, p++, q++;
  if(n == 0)
    return 0;
  return (uchar)*p - (uchar)*q;
}
```
