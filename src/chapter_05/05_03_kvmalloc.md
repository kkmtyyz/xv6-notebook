# 5.3. kvmalloc関数
この関数はカーネル用のページディレクトリとPDE、PTEを作成してそれに切り替える。  
4kBページングへの切り替えもここで行われる。

大域変数kpgdirに1ページ分のメモリを確保し、カーネル用のページディレクトリとする。  
ページディレクトリには次の4つのアドレス変換ができるようにPDEとPTEを作成する。  
  1. 仮想アドレス0x80000000～0x800FF000 → 物理アドレス0x0～0xff000
  2. 仮想アドレス0x80100000～0x80108000 → 物理アドレス0x100000～0x108000
  3. 仮想アドレス0x80109000～0x8DFFF000 → 物理アドレス0x109000～0xEDFFF000
  4. 仮想アドレス0xFE000000～ → 物理アドレス0xFE000000～

別な書き方をすると、それぞれ以下の領域のアドレスを変換する。

  1. I/Oスペース
  2. カーネルのテキストと読み込み専用データ
  3. カーネルのデータとメモリ
  4. メモリマップドI/Oを使用するデバイスのためのスペース

この変換は[「xv6 a simple, Unix-like teaching operating system」](https://pdos.csail.mit.edu/6.828/2018/xv6/book-rev11.pdf)の図2-2のマッピングを実現する。  
ページディレクトリの作成はsetupkvm関数で行い、PDEのbitの設定によりここで4MBページから4kBページに切り替わる。  
作成したページディレクトリをswitchkvm関数でcr3にロードする。

vm.c
```c
pde_t *kpgdir;  // for use in scheduler()

/* 略 */

void
kvmalloc(void)
{
  kpgdir = setupkvm();
  switchkvm();
}
```

## setupkvm関数
この関数はカーネル用のページディレクトリとPDE、PTEを作成する。  

PDEとPTEはkmap構造体の配列（kmap）を基に作成する。  
kmap構造体は、仮想アドレス（virt）を物理アドレス（phys\_start）にサイズ（phys\_end - phys\_start）分だけマップすることを示している。
また、その際にPTEの属性（perm）を設定する。  
変数dataはリンカスクリプトで作成されたシンボルで、text、rodata、stab、stabstrの後ろのページ境界に定義されている。今回は0x80109000に存在している。

`readelf -s kernel | grep data`
```
   411: 80109000     0 NOTYPE  GLOBAL DEFAULT    3 data
```

memlayout.h
```c
#define EXTMEM  0x100000            // Start of extended memory
#define PHYSTOP 0xE000000           // Top physical memory
#define DEVSPACE 0xFE000000         // Other devices are at high addresses

// Key addresses for address space layout (see kmap in vm.c for layout)
#define KERNBASE 0x80000000         // First kernel virtual address
#define KERNLINK (KERNBASE+EXTMEM)  // Address where kernel is linked
```

vm.c
```c
static struct kmap {
  void *virt;
  uint phys_start;
  uint phys_end;
  int perm;
} kmap[] = {
 { (void*)KERNBASE, 0,             EXTMEM,    PTE_W}, // I/O space
 { (void*)KERNLINK, V2P(KERNLINK), V2P(data), 0},     // kern text+rodata
 { (void*)data,     V2P(data),     PHYSTOP,   PTE_W}, // kern data+memory
 { (void*)DEVSPACE, DEVSPACE,      0,         PTE_W}, // more devices
};
```

setupkvm関数では、はじめにページディレクトリとしてkalloc関数で1ページ割り当て、0埋めする。  
デバイスのメモリマップドI/Oに使用されるアドレス（DEVSPACE）よりも、使用できる物理アドレスの上限（PHYSTOP）が大きかった場合、panicする。  
for文でkmap配列を走査し、kmap構造体で示された通りにPDEとPTEを作成する。PDEとPTEの作成にはmappages関数を使う。  
もしもPDEやPTEの作成に失敗した場合は、freevm関数でページディレクトリに含まれる全てのPDEとPTE、ページを解放する。

defs.h
```c
// number of elements in fixed-size array
#define NELEM(x) (sizeof(x)/sizeof((x)[0]))
```

vm.c
```c
pde_t*
setupkvm(void)
{
  pde_t *pgdir;
  struct kmap *k;

  if((pgdir = (pde_t*)kalloc()) == 0)
    return 0;
  memset(pgdir, 0, PGSIZE);
  if (P2V(PHYSTOP) > (void*)DEVSPACE)
    panic("PHYSTOP too high");
  for(k = kmap; k < &kmap[NELEM(kmap)]; k++)
    if(mappages(pgdir, k->virt, k->phys_end - k->phys_start,
                (uint)k->phys_start, k->perm) < 0) {
      freevm(pgdir);
      return 0;
    }
  return pgdir;
}
```


## kalloc関数
この関数はkmemのfreelistから1ページ割り当てる。

メモリが必要な時はこの先ずっとこの関数を使用して割り当てを行うので、AP起動後は排他制御が行われる。
kmemのfreelistの先頭を取ってきて返す。
freelistが空の場合、中身のないポインタを返す。

kalloc.c
```c
char*
kalloc(void)
{
  struct run *r;

  if(kmem.use_lock)
    acquire(&kmem.lock);
  r = kmem.freelist;
  if(r)
    kmem.freelist = r->next;
  if(kmem.use_lock)
    release(&kmem.lock);
  return (char*)r;
}
```

## mappages関数
この関数はページディレクトリに、仮想アドレスから物理アドレスへの変換が行えるPDEとPTEを作成する。  
関数を呼び出す際にページやPTEの範囲を気にする必要はなく、指定した物理アドレスの範囲を指定した仮想アドレスからアクセスできるようにマッピングしてくれる。

ページディレクトリ（pgdir）に、仮想アドレス（va）から物理アドレスの範囲（paからsizeバイト分）へ変換できるPDEとPTEを作成する。
PTEには属性（perm）を設定する。  

まず変換範囲の開始仮想アドレス（a）と終了仮想アドレス（last）を求める。
どちらもページサイズにアラインメントされている必要があるので、PGROUNDDOWNマクロを使用する。
PGROUNDDOWNマクロは、[「5.2. kinit1関数」](https://kkmtyyz.github.io/xv6-notebook/chapter_05/05_02_kinit1.html)で使用したPGROUNDUPマクロと同様の方法で、アドレスから前のページ境界のアドレスを計算する。
0x1000（PGSIZE）から1引いた0xFFFの否定0x000を論理積して4kBにアラインメントする。  

mmu.h
```
> 90 #define PGROUNDUP(sz)  (((sz)+PGSIZE-1) & ~(PGSIZE-1))
> 91 #define PGROUNDDOWN(a) (((a)) & ~(PGSIZE-1))
```

次にfor文で開始仮想アドレス（a）から、変換する最後の仮想アドレス（last）までを走査する。ループ毎の処理の流れは以下のよう。  
  1. 仮想アドレスのPTEを取得する。既にページディレクトリにPTEが存在する場合はそれを取得し、存在しない場合は作成する。これはwalkpgdir関数を用いて行う。  
  2. PTEが未使用であることを確認する。PTE\_Pフラグが0の場合は未使用、1の場合は使用されている。
  3. PTEに物理アドレスの上位20bitと属性bitをセットし、PTE\_Pフラグを立てる。

これで求めているアドレス変換が完成する。

vm.c
```c
static int
mappages(pde_t *pgdir, void *va, uint size, uint pa, int perm)
{
  char *a, *last;
  pte_t *pte;

  a = (char*)PGROUNDDOWN((uint)va);
  last = (char*)PGROUNDDOWN(((uint)va) + size - 1);
  for(;;){
    if((pte = walkpgdir(pgdir, a, 1)) == 0)
      return -1;
    if(*pte & PTE_P)
      panic("remap");
    *pte = pa | perm | PTE_P;
    if(a == last)
      break;
    a += PGSIZE;
    pa += PGSIZE;
  }
  return 0;
}
```


## walkpgdir関数
この関数はページディレクトリから、指定された仮想アドレスに対応するPTEを返す。  
PTEがすでに存在している場合はそれを返すが、無い場合は引数allocが1の場合に限り新たに作成する。  
また、PTEをPDEにセットする際に、読み書きフラグ（1bit）とユーザフラグ（2bit）を立てる。ページサイズフラグ（7bit）は立てないので、ページサイズは4kBとなる。  
PDEの構造と各bitの意味は[「Intel 64 and IA-32 architectures software developer's manual combined volumes: 1, 2A, 2B, 2C, 2D, 3A, 3B, 3C, 3D, and 4」（リンク8）](https://software.intel.com/content/www/us/en/develop/download/intel-64-and-ia-32-architectures-sdm-combined-volumes-1-2a-2b-2c-2d-3a-3b-3c-3d-and-4.html)の「Vol.3 4.3 32-BIT PAGING」に書いてある。

ある仮想アドレスが示すPDEのインデックスはPDXマクロで求めることができる。  
コメントに仮想アドレスの構造が示されているので分かりやすい。  
PDEを指すインデックスは仮想アドレスの22～31bitなので、22bit右シフトし、0b001111111111（0x3FF）で論理積を取ると求められる。
同様にPTEのインデックスはPTXマクロで求めることができる。  

mmu.h
```c
// A virtual address 'la' has a three-part structure as follows:
//
// +--------10------+-------10-------+---------12----------+
// | Page Directory |   Page Table   | Offset within Page  |
// |      Index     |      Index     |                     |
// +----------------+----------------+---------------------+
//  \--- PDX(va) --/ \--- PTX(va) --/

// page directory index
#define PDX(va)         (((uint)(va) >> PDXSHIFT) & 0x3FF)

// page table index
#define PTX(va)         (((uint)(va) >> PTXSHIFT) & 0x3FF)

/* 略 */

#define PTXSHIFT        12      // offset of PTX in a linear address
#define PDXSHIFT        22      // offset of PDX in a linear address
```

ページディレクトリから仮想アドレスの示すPDEを取得し、それが使用済みの場合（PTE\_P==1）はページテーブルを取得する。
ページテーブルの取得にはPTE\_ADDRマクロを使う。PDEの12～31bitがページテーブルのアドレスを示しており、ページテーブルは4kBでアラインメントされているため、下位12bitを0にするとアドレスが求まる。
このアドレスは物理アドレスなのでPTEへのアクセスはP2Vマクロで仮想アドレスに変換して行う。

mmu.h
```c
#define PTE_ADDR(pte)   ((uint)(pte) & ~0xFFF)
```

仮想アドレスが示すPDEが未使用（PTE\_P==0）かつ、引数allocが真の場合、新たにページテーブルを作成する。
ページテーブルとしてkalloc関数で1ページ分割り当て、内容を0埋めして初期化する。つまり全てのPTEの内容が0の状態。  
PDEにページテーブルのアドレス上位20bitと、読み書きフラグとユーザフラグをセットし、PTE\_Pフラグを立てる。繰り返しになるが、ページサイズフラグ（7bit）は立てないので、ページサイズは4kBとなる。

最後に、仮想アドレスが示すPTEをPTXマクロを使用してページテーブルから取得し、呼び出し元に返す。

vm.c
```c
static pte_t *
walkpgdir(pde_t *pgdir, const void *va, int alloc)
{
  pde_t *pde;
  pte_t *pgtab;

  pde = &pgdir[PDX(va)];
  if(*pde & PTE_P){
    pgtab = (pte_t*)P2V(PTE_ADDR(*pde));
  } else {
    if(!alloc || (pgtab = (pte_t*)kalloc()) == 0)
      return 0;
    // Make sure all those PTE_P bits are zero.
    memset(pgtab, 0, PGSIZE);
    // The permissions here are overly generous, but they can
    // be further restricted by the permissions in the page table
    // entries, if necessary.
    *pde = V2P(pgtab) | PTE_P | PTE_W | PTE_U;
  }
  return &pgtab[PTX(va)];
}
```


## switchkvm関数
この関数はカーネル用のページディレクトリをcr3にロードする。

呼び出し後からはアドレス変換がkpgdirを使用して行われる。  
lcr3関数は引数valを汎用レジスタのどれかに入れてcr3にセットする。  

vm.c
```c
void
switchkvm(void)
{
  lcr3(V2P(kpgdir));   // switch to the kernel page table
}
```

x86.h
```asm
static inline void
lcr3(uint val)
{
  asm volatile("movl %0,%%cr3" : : "r" (val));
}
```
