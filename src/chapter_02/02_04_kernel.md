# 2.4. ターゲットkernel
kernelの依存先としてentryotherとinitcodeがあるが、それらについては後ほど読むことにする。  
kernelは次のように作られる。  
  1. ターゲットkernelに依存している他のターゲットを作成する。
ひとつめが$(OBJS)になっていて、GNU makeの暗黙のルールにより、変数OJBSに記載されているオブジェクトが\*.cや\*.Sから作成される。
この時、vectors.oの作成のところでvectors.cもvectors.Sもないので、Makefile内のターゲットvectors.Sが実行され、vectors.plが実行される。
vectors.plはvector0からvector255までの割り込みハンドラが定義されたvectors.Sを作成する。
makeの暗黙のルールについてはマニュアル[「GNU make」（リンク6）](https://www.gnu.org/software/make/manual/make.html)の「10 Using Implicit Rules」に記載されている。
  2. 各オブジェクトをリンクし、バイナリファイルとしてkernelを作る。
Tオプションはリンカスクリプトを指定する。ここではkernel.ldを使う。kernel.ldを読むと、以下の設定をしている。  
このリンカスクリプトにより、作成されるkernelは仮想アドレス0x80100000で実行され、物理アドレス0x100000にロードされることが分かる。
  - 出力フォーマットをelf32-i386、出力アーキテクチャをi386、エントリポイントを\_startに設定。
  - textセクションの開始アドレスを0x80100000、ロード先アドレスを0x100000に設定し、オブジェクトファイルのtextセクション等を配置。
  - オブジェクト内でetextシンボルが定義されていない場合は、textセクションの後ろにetextシンボルを定義する。
  - textセクションの後ろに、オブジェクトファイルのrodataセクション等を含むrodataセクションを配置。
  - rodataセクションの後ろに、stabセクションを配置。オブジェクトファイルのstabセクションをシンボル\_\_STAB\_BEGIN\_\_と\_\_STAB\_END\_\_で挟む。
  - stabセクションの後ろに、stabstrセクションを配置。オブジェクトファイルのstabstrセクションをシンボル\_\_STABSTR\_BEGIN\_\_と\_\_STABSTR\_END\_\_で挟む。
  - stabセクションの後ろの次のページ境界に合うように（0x1000にアラインメント）し、シンボルdataを定義。
  - シンボルdataの後ろに、各オブジェクトファイルのdataセクションを含むdataセクションを配置。
  - dataセクションの後ろにシンボルedataを定義。
  - シンボルedataの後ろに、各オブジェクトファイルのbssセクションを含むbssセクションを配置。
  - シンボルendを定義。
  - 各オブジェクトファイルのeh\_frameセクションとnote.GNU-stackセクションをDISCARDで破棄。

  3. objdumpでディスアセンブリしてkernel.asmを作成。
  4. objdumpでkernelのシンボルテーブルを出力し、sedで加工してkernel.symを作成。

Makefile
```Makefile
kernel: $(OBJS) entry.o entryother initcode kernel.ld
	$(LD) $(LDFLAGS) -T kernel.ld -o kernel entry.o $(OBJS) -b binary initcode entryother
	$(OBJDUMP) -S kernel > kernel.asm
	$(OBJDUMP) -t kernel | sed '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > kernel.sym
```
