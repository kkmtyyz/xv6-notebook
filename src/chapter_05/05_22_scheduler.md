# 5.22. スケジューラとコンテキストスイッチ
scheduler関数は無限ループで次の2つを繰り返す。
1. sti関数で割り込みを有効化する
2. プロセステーブルを走査して実行可能状態（RUNNABLE）のプロセスにコンテキストスイッチする

最初にschedulerが実行されるとき、プロセステーブルの64個のプロセスの内、状態がRUNNABLEになっているのは[userinit関数](/chapter_05/05_21_userinit.md)で作成されたinitcode.Sのプロセスのみなので、それが実行されることになる。  
全てのプロセッサでschedulerが実行されており、プロセステーブルを触る可能性があるため、forループの前後でプロセステーブルのロックの取得と解放を行う。  
実行可能状態のプロセスを見つけた場合、switchuvm関数でコンテキストスイッチの準備を行う。  
準備ができたらプロセスを実行状態とし、swtch関数でコンテキストスイッチする。
コンテキストスイッチはTSSを用いたx86の機能を使うのではなく、スタックフレームの切り替えによって行う。  
その後再びプロセスからスケジューラにコンテキストスイッチされたとき、[switchkvm関数](/chapter_05/05_03_kvmalloc.md#switchkvm関数)でページディレクトリをカーネルのものに切り替え、cpu構造体のprocフィールドに0を代入する。

proc.c
```c
void
scheduler(void)
{
  struct proc *p;
  struct cpu *c = mycpu();
  c->proc = 0;
  
  for(;;){
    // Enable interrupts on this processor.
    sti();

    // Loop over process table looking for process to run.
    acquire(&ptable.lock);
    for(p = ptable.proc; p < &ptable.proc[NPROC]; p++){
      if(p->state != RUNNABLE)
        continue;

      // Switch to chosen process.  It is the process's job
      // to release ptable.lock and then reacquire it
      // before jumping back to us.
      c->proc = p;
      switchuvm(p);
      p->state = RUNNING;

      swtch(&(c->scheduler), p->context);
      switchkvm();

      // Process is done running for now.
      // It should have changed its p->state before coming back.
      c->proc = 0;
    }
    release(&ptable.lock);

  }
}
```

## コンテキストスイッチ
x86にはTSSディスクリプタを使用してコンテキストスイッチを行う機能があり、[「初めて読む486」（書籍2）](/ref_books.md)の8章「タスク」や、[「Intel 64 and IA-32 architectures software developer's manual combined volumes: 1, 2A, 2B, 2C, 2D, 3A, 3B, 3C, 3D, and 4」（リンク8）](https://software.intel.com/content/www/us/en/develop/download/intel-64-and-ia-32-architectures-sdm-combined-volumes-1-2a-2b-2c-2d-3a-3b-3c-3d-and-4.html)のVol.3 CHAPTER7「TASK MANAGEMENT」に記載があるが、xv6ではそれをフルに使わない。
TSSディスクリプタ自体はカーネルモードのスタックの情報とプロセスのIOMAPのために使用する。
これはLinuxカーネルも同様で、「詳解LINUXカーネル 第3版」（書籍3）の3.3.2「タスク状態セグメント」に記載がある。
xv6はタスク切り替えをスタックとret命令を使用して行う。

## switchuvm関数
この関数はコンテキストスイッチするプロセスのTSS構造体とそのディスクリプタを作成し、ltr命令でそれをロードする。  
また、[lcr3関数](/chapter_05/05_03_kvmalloc.md#switchkvm関数)でプロセスのページディレクトリをcr3にロードする。

TSSディスクリプタはcpuの[GDT](/chapter_05/05_06_seginit.md)の5番目にセットする。
x86のコンテキストスイッチ機能を使用する場合はプロセス切り替え時に、切り替え前のプロセスと切り替え後のプロセスの2つのTSSディスクリプタがGDTに必要だが、その機能を使用しないので毎回GDTの5番目にTSSディスクリプタを格納する。
TSSとTSSディスクリプタの構造はIntel-SDM（リンク8）のVol.3 Figure 7-2「32-Bit Task-State Segment (TSS)」とVol.3 Figure 7-3「TSS Descriptor」に書いてある。
TSSディスクリプタはmmu.hに定義されているSEG16マクロを使用して作成する。  
Intel-SDMのFigure 7-3によるとtypeフィールドの次のbit（segdesc構造体のsフィールド）は常に0。
segdesc構造体のコメントを見ると、このbitは0がシステム、1がアプリケーションを表すことがわかる。  
SEG16マクロの第二引数baseには、cpu構造体のtsフィールドのアドレスを渡す。
tsフィールドはtaskstate構造体で、Intel-SDMのFigure 7-2と同じ構造をしている。  
TSSではカーネルモードで使用するスタックセグメント（ss）とスタックポインタ（esp）、IOMAP（iomb）のみを使用する。
iombフィールドにはプロセスからin命令やout命令でアクセスを許可するポートを設定する。
bitが0のときにアクセスの許可を示し、ここでは0xFFFFで全てのbitを1に設定しているため、プロセスはどのポートへもアクセスできない。  
ltr命令でTSSディスクリプタをtrレジスタにロードする。
GDTは1エントリ8バイトなので、5を3bit左シフト（5\*2^3）することでTSSディスクリプタを指定する。  
プロセスのページディレクトリをcr3にロードし、仮想アドレス空間を切り替える。  
これでswitchuvm関数は終わりだが、まだeipやespの値は切り替わっていないため、scheduler関数に処理が戻る。

mmu.h
```c
#define SEG_TSS   5  // this process's task state

/* 略 */

#define SEG16(type, base, lim, dpl) (struct segdesc)  \
{ (lim) & 0xffff, (uint)(base) & 0xffff,              \
  ((uint)(base) >> 16) & 0xff, type, 1, dpl, 1,       \
  (uint)(lim) >> 16, 0, 0, 1, 0, (uint)(base) >> 24 }
```

vm.c
```c
void
switchuvm(struct proc *p)
{
  if(p == 0)
    panic("switchuvm: no process");
  if(p->kstack == 0)
    panic("switchuvm: no kstack");
  if(p->pgdir == 0)
    panic("switchuvm: no pgdir");

  pushcli();
  mycpu()->gdt[SEG_TSS] = SEG16(STS_T32A, &mycpu()->ts,
                                sizeof(mycpu()->ts)-1, 0);
  mycpu()->gdt[SEG_TSS].s = 0;
  mycpu()->ts.ss0 = SEG_KDATA << 3;
  mycpu()->ts.esp0 = (uint)p->kstack + KSTACKSIZE;
  // setting IOPL=0 in eflags *and* iomb beyond the tss segment limit
  // forbids I/O instructions (e.g., inb and outb) from user space
  mycpu()->ts.iomb = (ushort) 0xFFFF;
  ltr(SEG_TSS << 3);
  lcr3(V2P(p->pgdir));  // switch to process's address space
  popcli();
}
```

## swtch関数
この関数はスタックフレームの切り替えを行い、ret命令を使用してeipを切り替え先プロセスのリターンアドレスに移動させる。

この関数の動作を知るためにはGCCの関数呼び出し規約を知っている必要がある。
呼び出し規約は[「Guide: Function Calling Conventions」（リンク22）](http://www.delorie.com/djgpp/doc/ug/asm/calling.html)でいくつかの具体例とともに解説されている。
関数呼び出し時にはスタックに引数が右から左に順にpushされ、次にリターンアドレスがpushされる。
つまり引数が2つある場合は、第二引数がスタックにpushされ、次に第一引数がpushされ、最後にリターンアドレスがpushされる。
また、関数に入るときと出るときとでebx、esi、edi、ebpの値が変更されてはいけない。

swtch関数の動きとしては、espを切り替え先プロセスのスタックに切り替え、そこに積まれているリターンアドレスにret命令でeipを移動する。  
初めてのschedulerが呼ばれてから、プロセスへの切り替えの流れは以下のようになる。
1. swtch関数の第一引数にはスケジューラのコンテキスト（まだ0x0）、第二引数には[userinit関数](chapter_05/05_21_userinit.md)で作成されたproc構造体のcontextフィールドが渡される。
この時点でスタックには次のように値が積まれている。
添え字は積まれている順番で、括弧内は入っている値。
popすると2番目（リターンアドレス）が取り出されることになる。  
&nbsp;&nbsp;0: 第二引数（proc構造体のcontextフィールド）  
&nbsp;&nbsp;1: 第一引数（スケジューラのコンテキスト0x0）  
&nbsp;&nbsp;2: リターンアドレス（scheduler関数のswitchkvm関数を呼び出すアドレス）  
このとき、contextフィールドはcontext構造体だが、スタックと捉えることができる。
contextフィールドには次のように値が積まれている。  
&nbsp;&nbsp;0: eip（forkret関数のアドレス）  
&nbsp;&nbsp;1: ebp（0）  
&nbsp;&nbsp;2: ebx（0）  
&nbsp;&nbsp;3: esi（0）  
&nbsp;&nbsp;4: edi（0）  
2. スタックにebx、esi、edi、ebpの値を保存する。
スタックは次のようになる。  
&nbsp;&nbsp;0: 第二引数（proc構造体のcontextフィールド）  
&nbsp;&nbsp;1: 第一引数（スケジューラのコンテキスト0x0）  
&nbsp;&nbsp;2: リターンアドレス（scheduler関数のswitchkvm関数を呼び出すアドレス）  
&nbsp;&nbsp;3: ebp  
&nbsp;&nbsp;4: ebx  
&nbsp;&nbsp;5: esi  
&nbsp;&nbsp;6: edi  
3. スケジューラのcontext構造体がスタックを指すようにする（context構造体が完成する）。
4. espにproc構造体のcontextフィールドのアドレスを代入し、プロセスのスタックに切り替える。
5. プロセスのスタックからebx、esi、edi、ebpの値を復帰する。
スタックはeip（forkret関数のアドレス）のみが積まれた状態となる。
6. ret命令でforkret関数に移動し、プロセスの切り替えが終了する。

proc.h
```c
struct context {
  uint edi;
  uint esi;
  uint ebx;
  uint ebp;
  uint eip;
};
```

swtch.S
```asm
.globl swtch
swtch:
  movl 4(%esp), %eax
  movl 8(%esp), %edx

  # Save old callee-saved registers
  pushl %ebp
  pushl %ebx
  pushl %esi
  pushl %edi

  # Switch stacks
  movl %esp, (%eax)
  movl %edx, %esp

  # Load new callee-saved registers
  popl %edi
  popl %esi
  popl %ebx
  popl %ebp
  ret
```
