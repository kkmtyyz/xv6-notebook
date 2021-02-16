# 5.16. binit関数
この関数でバッファキャッシュを初期化を行う。

バッファキャッシュについては[5.12. バッファキャッシュとディスクの読み書き](https://kkmtyyz.github.io/xv6-notebook/chapter_05/05_12_bcache.html)で見た。  
キャッシュを走査するためのbcache内のheadフィールドの前（prev）と後（next）のアドレスをheadフィールドの値で初期化する。  
そしてforループでバッファキャッシュの全エントリ（30個）を初期化し、buf配列が循環するようにリンクさせる。
ループが終わると、headのnextはbuf配列の29番目を指すようになり、prevは0番目を指すようになる。
29番目のprevと0番目のnextはheadを指す。  
headからnextを辿ると、buf配列の29, 28, 27 ... 0, headとなる。  
headからprevを辿ると、buf配列の0, 1, 2 ... 29, headとなる。  

bio.c
```c
void
binit(void)
{
  struct buf *b;

  initlock(&bcache.lock, "bcache");

//PAGEBREAK!
  // Create linked list of buffers
  bcache.head.prev = &bcache.head;
  bcache.head.next = &bcache.head;
  for(b = bcache.buf; b < bcache.buf+NBUF; b++){
    b->next = bcache.head.next;
    b->prev = &bcache.head;
    initsleeplock(&b->lock, "buffer");
    bcache.head.next->prev = b;
    bcache.head.next = b;
  }
}
```

