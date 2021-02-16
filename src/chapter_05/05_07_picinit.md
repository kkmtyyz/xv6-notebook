# 5.7. picinit関数
この関数はMPに対応していない古いPICでの割り込みを無効化する。

outb関数でポート0x20と0xA1に0xFFを書き込んでいる。
[「OSDev I/O Ports」（リンク10）](https://ftp.gnu.org/old-gnu/Manuals/ld-2.9.1/html_mono/ld.html)によると、ポート0x21はThe first Programmable Interrupt Controllerで、ポート0xA1はThe second PIC。
[「OSDev 8259 PIC」（リンク16）](https://wiki.osdev.org/PIC)の「Disabling」によると、LAPICとIOAPICを使用する場合はPICを無効化しなければいならず、その方法として0xFFを設定する。

picirq.c
```c
#define IO_PIC1         0x20    // Master (IRQs 0-7)
#define IO_PIC2         0xA0    // Slave (IRQs 8-15)

// Don't use the 8259A interrupt controllers.  Xv6 assumes SMP hardware.
void
picinit(void)
{
  // mask all interrupts
  outb(IO_PIC1+1, 0xFF);
  outb(IO_PIC2+1, 0xFF);
}
```
