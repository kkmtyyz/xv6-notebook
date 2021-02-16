# 5.4. mpinit関数
この関数は大域変数lapicにLAPICへのアクセスアドレスを設定、各cpu構造体にLAPICのIDを設定、大域変数ioapicidにIOAPICのIDを設定する。

この関数ではマルチプロセッサに関する情報を取得する。  
xv6では使用できる最大CPU数をNCPUとして8に定義している。システムで実際に使用可能なCPU数は、大域変数ncpuに保持する。  
また、CPUに関する情報を持つ構造体としてcpu構造体が定義されており、配列cpusとして保持している。  

param.h
```c
#define NCPU          8  // maximum number of CPUs
```

proc.h
```c
struct cpu {
  uchar apicid;                // Local APIC ID
  struct context *scheduler;   // swtch() here to enter scheduler
  struct taskstate ts;         // Used by x86 to find stack for interrupt
  struct segdesc gdt[NSEGS];   // x86 global descriptor table
  volatile uint started;       // Has the CPU started?
  int ncli;                    // Depth of pushcli nesting.
  int intena;                  // Were interrupts enabled before pushcli?
  struct proc *proc;           // The process running on this cpu or null
};
```

mp.c
```c
struct cpu cpus[NCPU];
int ncpu;
```

マルチプロセッサに関する構造体が定義されているmp.hの1行目に[「MultiProcessor Specification Version 1.4」（リンク14）](https://pdos.csail.mit.edu/6.828/2008/readings/ia32/MPspec.pdf)を見るように書いてある。  
このMP仕様書によると、プロセッサは1つのBSP（ブートストラッププロセッサ）とその他のAP（アプリケーションプロセッサ）とに分けられ、起動時はBSPが動いている。
各プロセッサはLAPICを有しており、ICC（割り込みコントローラバス）を通してIOAPICと接続されている。
LAPICに振られているIDでプロセッサを区別することが可能。
起動時には全てのAPのLAPICと、全てのIOAPICが無効化されているため、それらを初期化しなければならない。  
また、割り込みモードとして、PICモード、仮想配線モード、Symmetric I/Oモードの3つがある。
システムはPICモードか仮想配線モードで開始され、マルチプロセッサでの割り込みを行うためにSymmetric I/Oモードに移行する必要がある。
MP仕様書の「Table 4-1. MP Floating Pointer Structure Fields」にIMCRがない場合は仮想配線モードが実装されていると書いてある。
IMCR（Interrupt Mode Configuration Register）はPICモードの時に使用されるレジスタ。
Symmetric I/Oモードへの移行方法はMP仕様書の「3.6.2.3 Symmetric I/O Mode」によると、IMCRがある場合はそこに0x1を書き込み、IOAPICのリダイレクションテーブルのエントリを有効化することで行える。  

メモリ上には、マルチプロセッサに関する情報を取得するために、MPフローティングポインタ構造体と、MP設定テーブルが用意されている（後者はオプション）。  
MPフローティングポインタ構造体はシステムのMPに関する基本的な情報を持っており、以下のいずれかの場所にある。  
<ol type='a'>
  <li>EBDA（拡張BIOSデータ領域）の最初の1キロバイト</li>
  <li>システムベースメモリの最後の1キロバイト以内のどこか</li>
  <li>0xF0000から0xFFFFFまでのBIOS ROMアドレスのどこか</li>
</ol>

MP設定テーブルの開始アドレスは、MPフローティングポインタ構造体のPHYSICAL ADDRESS POINTERフィールド（4バイト目）が持っている。このフィールドの値が0の場合は、MP設定テーブルは存在しない。  
MP設定テーブルはひとつのヘッダ部分と、複数のエントリ部分に分かれている。ヘッダにはMP設定テーブルの基本的な情報を持っている。  
基本的なエントリとその構造はMP仕様書の「Table 4-3. Base MP Configuration Table Entry types」に定義されており、以下の5つのエントリがある。各エントリはタイプコード（0バイト目）で識別することが可能。  
  1. Processor: 0（タイプコード）
  2. Bus: 1
  3. I/O APIC: 2
  4. I/O Interrupt Assignment: 3
  5. Local Interrupt Assignment: 4
  
このエントリとタイプコードに合わせるようにmp.hに定義がある。

mp.h
```c
#define MPPROC    0x00  // One per processor
#define MPBUS     0x01  // One per bus
#define MPIOAPIC  0x02  // One per I/O APIC
#define MPIOINTR  0x03  // One per bus interrupt source
#define MPLINTR   0x04  // One per system interrupt source
```

mpinit関数は、まずMPフローティングポインタ構造体と、MP設定テーブルを取得する。取得にはmpconfig関数を使う。  
MP設定テーブルからLAPICへのアクセスアドレスを取得し、大域変数lapicに設定する。
アドレスはADDRESS OF LOCAL APICフィールド（36バイト目）から取得可能。
LAPICはプロセッサに内臓されているため、各プロセッサが個別に持っているが、MP仕様書によると、各プロセッサは自分のLAPICにアクセスするために同じアドレスにアクセスする。アドレスは同じでもアクセスは自分のLAPICへのものになる。

for文でMP設定テーブルのエントリをポインタpで走査する。  
エントリのタイプコード（1バイト目）により処理内容が分かれる。未知のエントリが検出された場合はpanic。  

**Processerの場合:**  
大域変数cpus[ncpu]のapicidフィールドにLAPIC IDを設定する。LAPIC IDは、エントリのLOCAL APIC IDフィールド（1バイト目）にある。  
次のプロセッサエントリのためにncpuをインクリメントする。  
pにエントリのサイズを加えて、ループを継続する。

**I/O APICの場合:**  
大域変数ioapicidにIOAPIC IDを設定する。IOAPIC IDは、エントリのI/O APIC IDフィールド（1バイト目）にある。  
pにエントリのサイズを加えて、ループを継続する。

**Bus, I/O Interrupt Assignment, Local Interrupt Assignmentの場合:**  
何もしない。  
pにエントリのサイズを加えて、ループを継続する。

全てのエントリの処理が終わった時点で、以下の大域変数が設定されている。  
lapic: LAPICへのアクセスアドレスが設定されている。  
ncpu: システムで使用できるCPU数が設定されている。  
cpus配列: ncpu分のapicidフィールドに各LAPIC IDが設定されている。  
ioapicid: IOAPIC IDが設定されている。  

mpinit関数の最後の処理、IMCR（Interrupt Mode Configuration Register）がシステムに実装されている場合、Symmetric I/Oモードへの移行準備として、そこに0x1を書き込む。  
IMCRの有無は、MPフローティングポインタ構造体のMP FEATURE INFORMATION BYTE 2フィールド（12バイト目）の7bit目が1であればIMCR有り、0なら無しと判断できる。
0から6bit目まではMP仕様で予約されている。
MP仕様書の「3.6.2.1 PIC Mode」によると、IMCRへはポート0x22に0x70を書き込む事でアクセスできる。データの読み書きはポート0x23に行う。
IMCRの内容をそのまま0bitを1にする。

mp.c
```c
void
mpinit(void)
{
  uchar *p, *e;
  int ismp;
  struct mp *mp;
  struct mpconf *conf;
  struct mpproc *proc;
  struct mpioapic *ioapic;

  if((conf = mpconfig(&mp)) == 0)
    panic("Expect to run on an SMP");
  ismp = 1;
  lapic = (uint*)conf->lapicaddr;
  for(p=(uchar*)(conf+1), e=(uchar*)conf+conf->length; p<e; ){
    switch(*p){
    case MPPROC:
      proc = (struct mpproc*)p;
      if(ncpu < NCPU) {
        cpus[ncpu].apicid = proc->apicid;  // apicid may differ from ncpu
        ncpu++;
      }
      p += sizeof(struct mpproc);
      continue;
    case MPIOAPIC:
      ioapic = (struct mpioapic*)p;
      ioapicid = ioapic->apicno;
      p += sizeof(struct mpioapic);
      continue;
    case MPBUS:
    case MPIOINTR:
    case MPLINTR:
      p += 8;
      continue;
    default:
      ismp = 0;
      break;
    }
  }
  if(!ismp)
    panic("Didn't find a suitable machine");

  if(mp->imcrp){
    // Bochs doesn't support IMCR, so this doesn't run on Bochs.
    // But it would on real hardware.
    outb(0x22, 0x70);   // Select IMCR
    outb(0x23, inb(0x23) | 1);  // Mask external interrupts.
  }
}
```

## mpconfig関数
この関数はメモリからMP設定テーブルを取得する。

MPフローティングポインタ構造体をmpconfig関数で取得し、そのPHYSICAL ADDRESS POINTERフィールド（4バイト目）からMP設定テーブルのアドレスを取得する。  
MP設定テーブルのSIGNATUREフィールド（0バイト目）が “PCMP” であることを確認し、M確かにMP設定テーブルであることを確認する。
準拠しているMP仕様のバージョンを確認する。これはMP設定テーブルのSPEC\_REVフィールド（9バイト目）から確認でき、[「MultiProcessor Specification Version 1.4」（リンク14）](https://pdos.csail.mit.edu/6.828/2008/readings/ia32/MPspec.pdf)の「Table 4-1. MP Floating Pointer Structure Fields」によると0x1がバージョン1.1、0x4がバージョン1.4を示す。  
チェックサムの計算を行う。MP設定テーブル全体の値を1バイト単位で合計し、結果が0となることでMP設定テーブルに誤りがないことを確認する。CHECKSUMフィールド（10バイト目）の値は結果が0になるような値が設定されている。  
引数pmpにMPフローティングポインタ構造体を代入し、MP設定テーブルを呼び出し元に返す。

mp.c
```c
static uchar
sum(uchar *addr, int len)
{
  int i, sum;

  sum = 0;
  for(i=0; i<len; i++)
    sum += addr[i];
  return sum;
}

/* 略 */

static struct mpconf*
mpconfig(struct mp **pmp)
{
  struct mpconf *conf;
  struct mp *mp;

  if((mp = mpsearch()) == 0 || mp->physaddr == 0)
    return 0;
  conf = (struct mpconf*) P2V((uint) mp->physaddr);
  if(memcmp(conf, "PCMP", 4) != 0)
    return 0;
  if(conf->version != 1 && conf->version != 4)
    return 0;
  if(sum((uchar*)conf, conf->length) != 0)
    return 0;
  *pmp = mp;
  return conf;
}
```

## mpsearch関数
この関数はメモリからMPフローティングポインタ構造体を取得する。

[「MultiProcessor Specification Version 1.4」（リンク14）](https://pdos.csail.mit.edu/6.828/2008/readings/ia32/MPspec.pdf)の「3.9 Support for Fault-resilient Booting」に記載されているaからcの方法を順に試し、MPフローティングポインタ構造体を探す。  
<ol type='a'>
  <li>EBDA（拡張BIOSデータ領域）の最初の1キロバイト</li>
  <li>システムベースメモリの最後の1キロバイト以内のどこか</li>
  <li>0xF0000から0xFFFFFまでのBIOS ROMアドレスのどこか</li>
</ol>

各範囲の走査にはmpsearch1関数を使用する。

まず最初のif文で方法aを試す。  
BIOSからブートローダに移る際のx86のメモリマップは[「OSDev Memory Map (x86)」（リンク15）](https://wiki.osdev.org/Memory_Map_(x86))に記載されている。
それによると、BDAは0x0400から始まり、0x040Eから2バイト分には、4bit右シフトされたEBDAの物理アドレスが入っている。
なので、0x40Fの内容を上位8bit、0x40Eの内容を下位8bitとして、4bit左シフトすると、EBDAの物理アドレスが求められる。

次にelse文で方法bを試す。  
システムベースメモリというのはmpsearch関数を読む限りではEBDAの開始アドレスまでを指す。
BDA内、0x0413から2バイト分には、EBDAまでのキロバイト数が記してある。
なので、0x414の内容を上位8bit、0x413の内容を下位8bitとして、1024掛けるとEBDAまでのバイト数になる。
EBDAの手前1キロバイト以内を探す。

最後にreturn文で方法cを試す。

mp.c
```c
static struct mp*
mpsearch(void)
{
  uchar *bda;
  uint p;
  struct mp *mp;

  bda = (uchar *) P2V(0x400);
  if((p = ((bda[0x0F]<<8)| bda[0x0E]) << 4)){
    if((mp = mpsearch1(p, 1024)))
      return mp;
  } else {
    p = ((bda[0x14]<<8)|bda[0x13])*1024;
    if((mp = mpsearch1(p-1024, 1024)))
      return mp;
  }
  return mpsearch1(0xF0000, 0x10000);
}
```

mpsearch1関数では与えられた物理アドレスと長さ分の範囲を走査し、MPフローティングポインタ構造体を探す。  
MP仕様書の「Table 4-1. MP Floating Pointer Structure Fields」によると、SIGNATUREフィールド（0バイト目）に  ”\_MP\_” が入っているため、それを頼りに同構造体を探すことができる。
また、CHECKSUMフィールド（10バイト目）により、見つけた領域がMPフローティングポインタ構造体であるか否かを断定できる。
チェックサムの計算方法は[「mpconfig関数」](/chapter_05/05_04_mpinit.md#mpconfig関数)のときと同様。

mp.c
```c
// Look for an MP structure in the len bytes at addr.
static struct mp*
mpsearch1(uint a, int len)
{
  uchar *e, *p, *addr;

  addr = P2V(a);
  e = addr+len;
  for(p = addr; p < e; p += sizeof(struct mp))
    if(memcmp(p, "_MP_", 4) == 0 && sum(p, sizeof(struct mp)) == 0)
      return (struct mp*)p;
  return 0;
}
```


## memcmp関数
引数v1と引数v2の値をnバイトまで比較し、等しい場合は0を、等しくない場合は一致しなかった値のv1 - v2の値を返す。

string.c
```c
int
memcmp(const void *v1, const void *v2, uint n)
{
  const uchar *s1, *s2;

  s1 = v1;
  s2 = v2;
  while(n-- > 0){
    if(*s1 != *s2)
      return *s1 - *s2;
    s1++, s2++;
  }

  return 0;
}
```
