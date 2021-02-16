# 5.17. fileinit関数
この関数はファイルテーブル構造体のロックを初期化する。

ファイルテーブル構造体は大域変数ftableとして定義されており、fileフィールドにはオープンしているファイルを持つ。
要素数100なのでオープンできるのは最大100ファイル。

param.h
```c
#define NFILE       100  // open files per system
```

file.c
```c
struct {
  struct spinlock lock;
  struct file file[NFILE];
} ftable;

void
fileinit(void)
{
  initlock(&ftable.lock, "ftable");
}
```

## file構造体
各フィールドは名前の通り。  
タイプに合わせてpipeフィールドかinodeフィールドに各構造体を持つ。  
iノードの場合はoffフィールドにファイル内でのオフセットを持つ。

file.h
```c
struct file {
  enum { FD_NONE, FD_PIPE, FD_INODE } type;
  int ref; // reference count
  char readable;
  char writable;
  struct pipe *pipe;
  struct inode *ip;
  uint off;
};
```

## pipe構造体
dataフィールドはパイプで流すデータを保持する。  
他のフィールドはコメントの通り。  

data, nread, nwriteフィールドは、[consoleintr関数とconsoleread関数](/chapter_05/05_09_consoleinit.md#consoleintr関数)で使用されるinput構造体のbuf, r, wフィールドと同様の使い方をする。
つまりnreadとnwriteは増加し続け、dataにはnreadとnwriteをPIPESIZEで割った余りをインデックスとしてアクセスする。

pipe.c
```c
#define PIPESIZE 512

struct pipe {
  struct spinlock lock;
  char data[PIPESIZE];
  uint nread;     // number of bytes read
  uint nwrite;    // number of bytes written
  int readopen;   // read fd is still open
  int writeopen;  // write fd is still open
};
```
