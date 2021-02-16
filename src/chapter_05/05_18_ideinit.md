# 5.18. ideinit関数
この関数ではIDEの割り込みの有効化と、ディスク1の存在確認を行う。

ide.cに宣言されている静的なint型の変数havedisk1が、ディスク1の有無を示すフラグとなっており、値が1場合にディスク1があることを示す。  

IDEからの割り込みを[ioapicenable関数](/chapter_05/05_09_consoleinit.md#ioapicenable関数)で有効化する。
IRQ番号は[「OSDev Interrupt」（リンク29）](https://wiki.osdev.org/Interrupts)の表「Standard ISA IRQs」をによると、14番がPrimary ATA Hard Disk、と15番がSecondary ATA Hard Diskとなっている。
ここでは14番（IRQ\_IDE）からの割り込みを有効化する。
割り込み先はcpuid（LAPIC ID）が最も大きいcpuに設定する。
cpu数を示す大域変数ncpuは[mpinit関数](/chapter_05/05_04_mpinit.md)で設定した。

[idewait関数](/chapter_05/05_12_bcache.md#ディスクの読み書きiderw関数)でディスク0がビジーでもエラーでもなくなるまで待つ。  
ディスク1の有無を確認するために、読み書きを行うディスクをディスク1に切り替える。
[「OSDev ATA PIO Mode」（リンク23）](https://wiki.osdev.org/ATA_PIO_Mode)によると、IDEコントローラのdrive/headレジスタ（0x1f6）の4bitを1にすることでディスク1に切り替えられる。
また、5～7bitは101で固定のようだが、ここでは0xe0で全て1に設定している。  
ディスク1のステータスレジスタ（0x1f7）から0以外の値が読み取れるまで、for文で1000回ループする。
エラーになっていようがレディになっていようがとりあえず0以外が返ってくればディスク1が存在することは確認できる。  
ディスク1の存在確認完了後、読み書きするディスクをディスク0に戻す。

traps.h
```c
#define IRQ_IDE         14
```

ide.c
```c
static int havedisk1;

/* 略 */

void
ideinit(void)
{
  int i;

  initlock(&idelock, "ide");
  ioapicenable(IRQ_IDE, ncpu - 1);
  idewait(0);

  // Check if disk 1 is present
  outb(0x1f6, 0xe0 | (1<<4));
  for(i=0; i<1000; i++){
    if(inb(0x1f7) != 0){
      havedisk1 = 1;
      break;
    }
  }

  // Switch back to disk 0.
  outb(0x1f6, 0xe0 | (0<<4));
}
```
