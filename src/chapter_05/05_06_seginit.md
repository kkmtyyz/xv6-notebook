# 5.6. seginit関数
この関数はGDTを作成し、ロードする。

ここまでは[「3.1. bootasm.S」](/chapter_03/03_01_bootasm.md)で作成した3つのエントリを持つGDTを使用してきた。  
ここではユーザ空間用のセグメントディスクリプタを持った新しいGDTを作成する。

セグメントディスクリプタの構造は[「Intel 64 and IA-32 architectures software developer's manual combined volumes: 1, 2A, 2B, 2C, 2D, 3A, 3B, 3C, 3D, and 4」（リンク8）](https://software.intel.com/content/www/us/en/develop/download/intel-64-and-ia-32-architectures-sdm-combined-volumes-1-2a-2b-2c-2d-3a-3b-3c-3d-and-4.html)の「Vol.3 3.4.5 Segment Descriptors」に書いてある。

関数を実行しているcpu構造体のgdtフィールドに以下の4つのセグメントディスクリプタを作成する。0番目はNULLディスクリプタなので初期化は不要。  

&nbsp;&nbsp;1番目: カーネル用のコードセグメントディスクリプタ  
&nbsp;&nbsp;2番目: カーネル用のデータセグメントディスクリプタ  
&nbsp;&nbsp;3番目: ユーザ用のコードセグメントディスクリプタ  
&nbsp;&nbsp;4番目: ユーザ用のデータセグメントディスクリプタ  

4つともベースアドレスは0x0、セグメントリミットは0xFFFFF。  
Gフラグが1なので、セグメントリミットはページサイズ倍（4kB)され0xFFFFF000（4GB）となる。  
タイプは4つとも読み書き可能、コードセグメントディスクリプタのみ実行可能にもなっている。  
DPL（Descriptor Privilege Level）はカーネル用は0、ユーザ用は3。  

全てのセグメントの範囲が0から4GBまでで重なっている。
メモリ管理にはページング機構を使用するが、セグメント機構は無効化できないため、こうして4GBまでをセグメント機構的になんにでも使えるようにしている。  
cpu構造体のgdtフィールドは要素数が6（定数NSEGS）に定義されているため、GDTにはエントリが6つあることになる。
0から4番目まではここで作成するが、5番目はプロセスのコンテキストスイッチを行う際にTSS（タスク状態セグメント）ディスクリプタとして作成することになる。

作成したGDTをlgdt関数でgdtレジスタにロードする。  
このとき、ecsレジスタの値は更新不要。
[「3.1. bootasm.S」](/chapter_03/03_01_bootasm.md#プロテクトモードへの切り替え)ではファージャンプを行うことでecsの設定を行っているが、コードセグメントディスクリプタのインデックスがここで作成したGDTでも変わらず1番目（8バイト目）であるため、再設定が不要。

mmu.h
```c
#define SEG_KCODE 1  // kernel code
#define SEG_KDATA 2  // kernel data+stack
#define SEG_UCODE 3  // user code
#define SEG_UDATA 4  // user data+stack
#define SEG_TSS   5  // this process's task state

/* 略 */

#define SEG(type, base, lim, dpl) (struct segdesc)    \
{ ((lim) >> 12) & 0xffff, (uint)(base) & 0xffff,      \
  ((uint)(base) >> 16) & 0xff, type, 1, dpl, 1,       \
  (uint)(lim) >> 28, 0, 0, 1, 1, (uint)(base) >> 24 }

/* 略 */

#define STA_X       0x8     // Executable segment
#define STA_W       0x2     // Writeable (non-executable segments)
#define STA_R       0x2     // Readable (executable segments)
```

vm.c
```c
void
seginit(void)
{
  struct cpu *c;

  // Map "logical" addresses to virtual addresses using identity map.
  // Cannot share a CODE descriptor for both kernel and user
  // because it would have to have DPL_USR, but the CPU forbids
  // an interrupt from CPL=0 to DPL=3.
  c = &cpus[cpuid()];
  c->gdt[SEG_KCODE] = SEG(STA_X|STA_R, 0, 0xffffffff, 0);
  c->gdt[SEG_KDATA] = SEG(STA_W, 0, 0xffffffff, 0);
  c->gdt[SEG_UCODE] = SEG(STA_X|STA_R, 0, 0xffffffff, DPL_USER);
  c->gdt[SEG_UDATA] = SEG(STA_W, 0, 0xffffffff, DPL_USER);
  lgdt(c->gdt, sizeof(c->gdt));
}
```

作成されたセグメントディスクリプタをgdbで確認すると以下のようになっている。

```
$1 = {lim_15_0 = 0x0, base_15_0 = 0x0, base_23_16 = 0x0, type = 0x0, s = 0x0, dpl = 0x0, p = 0x0, lim_19_16 = 0x0, avl = 0x0, rsv1 = 0x0, db = 0x0, g = 0x0, base_31_24 = 0x0}
$2 = {lim_15_0 = 0xffff, base_15_0 = 0x0, base_23_16 = 0x0, type = 0xa, s = 0x1, dpl = 0x0, p = 0x1, lim_19_16 = 0xf, avl = 0x0, rsv1 = 0x0, db = 0x1, g = 0x1, base_31_24 = 0x0}
$3 = {lim_15_0 = 0xffff, base_15_0 = 0x0, base_23_16 = 0x0, type = 0x2, s = 0x1, dpl = 0x0, p = 0x1, lim_19_16 = 0xf, avl = 0x0, rsv1 = 0x0, db = 0x1, g = 0x1, base_31_24 = 0x0}
$4 = {lim_15_0 = 0xffff, base_15_0 = 0x0, base_23_16 = 0x0, type = 0xa, s = 0x1, dpl = 0x3, p = 0x1, lim_19_16 = 0xf, avl = 0x0, rsv1 = 0x0, db = 0x1, g = 0x1, base_31_24 = 0x0}
$5 = {lim_15_0 = 0xffff, base_15_0 = 0x0, base_23_16 = 0x0, type = 0x2, s = 0x1, dpl = 0x3, p = 0x1, lim_19_16 = 0xf, avl = 0x0, rsv1 = 0x0, db = 0x1, g = 0x1, base_31_24 = 0x0}
```

## cpuid関数
この関数は、この関数を実行しているプロセッサのcpuidを返す。

ここで言うcpuidとは、cpus配列のインデックスのこと。  
各プロセッサのcpu構造体はcpus配列のエントリとして保持しており、mycpu関数はそれを実行しているプロセッサのエントリを返してくれる。  
そのエントリのアドレスからcpus配列の先頭アドレスを引けば、cpus配列内でのインデックスになる。

proc.c
```c
> 30 int
> 31 cpuid() {
> 32   return mycpu()-cpus;
> 33 }
```

## mycpu関数
この関数は、この関数を実行しているプロセッサのcpu構造体をcpus配列から探して返す。

eflagsレジスタの値を取得し、Interrupt enableビット（9bit）を確認して、割り込みが無効になっていることを確認する。
0が無効で1が有効。
eflagsレジスタの各bitの役割は[「Intel 64 and IA-32 architectures software developer's manual combined volumes: 1, 2A, 2B, 2C, 2D, 3A, 3B, 3C, 3D, and 4」（リンク8）](https://software.intel.com/content/www/us/en/develop/download/intel-64-and-ia-32-architectures-sdm-combined-volumes-1-2a-2b-2c-2d-3a-3b-3c-3d-and-4.html)の「Vol.3 2.3 SYSTEM FLAGS AND FIELDS IN THE EFLAGS REGISTER」に書いてある。

cpus配列をfor文で走査し、apicidフィールドの値が関数を実行しているプロセッサのLAPIC IDと等しいエントリを探す。
各cpu構造体のapicidフィールドは[「5.4. mpinit関数」](/chapter_05/05_04_mpinit.md)で設定済み。

mmu.h
```c
#define FL_IF           0x00000200      // Interrupt Enable
```

proc.c
```c
struct cpu*
mycpu(void)
{
  int apicid, i;
  
  if(readeflags()&FL_IF)
    panic("mycpu called with interrupts enabled\n");
  
  apicid = lapicid();
  // APIC IDs are not guaranteed to be contiguous. Maybe we should have
  // a reverse map, or reserve a register to store &cpus[i].
  for (i = 0; i < ncpu; ++i) {
    if (cpus[i].apicid == apicid)
      return &cpus[i];
  }
  panic("unknown apicid\n");
}
```

## readeflags関数
この関数はeflagsレジスタの値を取得する。  
pushfl命令はeflagsレジスタの内容をスタックに積む命令。

x86.h
```c
static inline uint
readeflags(void)
{
  uint eflags;
  asm volatile("pushfl; popl %0" : "=r" (eflags));
  return eflags;
}
```

## lapic関数
この関数は、この関数を実行しているプロセッサのLAPIC IDを取得する。

LAPIC IDは [「Intel 64 and IA-32 architectures software developer's manual combined volumes: 1, 2A, 2B, 2C, 2D, 3A, 3B, 3C, 3D, and 4」（リンク8）](https://software.intel.com/content/www/us/en/develop/download/intel-64-and-ia-32-architectures-sdm-combined-volumes-1-2a-2b-2c-2d-3a-3b-3c-3d-and-4.html)の「Vol.3 10.4.6 Local APIC ID」によると、Local APIC ID Register（0x20）の24～31bitから得られる。  
LAPICへのアクセスについては[「5.4. mpinit関数」](/chapter_05/05_04_mpinit.md)に書いた。

lapic.c
```c
int
lapicid(void)
{
  if (!lapic)
    return 0;
  return lapic[ID] >> 24;
}
```
