# 2.2. Makefile内の変数
ターゲットbootblockとkernelを見る前に、Makefile内の変数を確認する。  
変数CCがgccなのでコンパイラにはgccを使う。  
変数CFLAGSは後ろの方が環境によって変わるようなので、makeを実行して中身を確認すると以下のようになっていることが分かる。  
`CFLAGS = -fno-pic -static -fno-builtin -fno-strict-aliasing -O2 -Wall -MD -ggdb -m32 -Werror -fno-omit-frame-pointer -fno-stack-protector`  
各オプションの意味はマニュアル[「Using the GNU Compiler Collection (GCC)」（リンク3）](https://gcc.gnu.org/onlinedocs/gcc-6.5.0/gcc/)を見るとだいたいわかる。
知りたいオプションをOption Summaryページでページ内検索して、各ページに飛ぶのと楽。  

| Option                 | 意味                                                                                                                                    |
|------------------------|---------------------------------------------------------------------------------------------------------------------------------------|
| fno-pic                | マニュアルにこのオプションの記載はないが、fpicオプションはあるのでその否定だとわかる。出力を位置独立コード（Position Independent Code）にしない。位置独立コードに関しては[「リンカ・ローダ 実践開発テクニック」（書籍1）](ref_books.md)に詳しく載っている。 |
| static                 | 動的ライブラリを静的リンクする。                                                                                                                      |
| fno-builtin            | 組み込み関数を使わないようにする。                                                                                                                     |
| O2                     | レベル2の最適化オプションを有効にする。                                                                                                                  |
| Wall                   | 警告オプションを有効にする。                                                                                                                        |
| MD                     | 拡張子「\*.d」ファイルを作成し、依存関係にあるファイル名を書き込んでくれる。                                                                                                   |
| ggdb                   | gdbで使用するデバッグ情報を作る。                                                                                                                  |
| m32                    | int, long, ポインタを32bitにする。                                                                                                             |
| Werror                 | 全ての警告をエラーにする。                                                                                                                         |
| fno-omit-frame-pointer | fomit-frame-pointerオプションの否定。関数呼び出しにて必ずベースポインタを使う。                                             |
| fno-stack-protector    | fstack-protectorオプションの否定。バッファオーバーフローやスタックスマッシング攻撃をチェックするコードを追加しない。                                          |

変数LDがldなのでリンカはldを使うことが分かる。  
変数LDFLAGSもCFLAGS同様にmakeを実行して出力を確認すると以下のようになっていることが分かる。  
`LDFLAGS = -m    elf_i386`  
オプションの意味はマニュアル[「Using ld The GNU linker ld version 2 January 1994」（リンク4）](https://ftp.gnu.org/old-gnu/Manuals/ld-2.9.1/html_mono/ld.html)で確認できる。このマニュアルはIndexからオプションが簡単に探せる。

| Option | 意味                                                                                |
|--------|-----------------------------------------------------------------------------------|
| m      | 指定したリンカをエミュレートする。ここではelf\_i386をエミュレート。elfについては[「リンカ・ローダ 実践開発テクニック」（書籍1）](ref_books.md)に詳しく記載されている。 |

変数OBJDUMPはobjdump。  
変数OBJCOPYはobjcopy。マニュアル[「3 objcopy」（リンク5）](https://sourceware.org/binutils/docs-2.35/binutils/objcopy.html#objcopy)によると、Oオプションで出力ファイルのフォーマットを指定できる。また、出力フォーマットにバイナリを指定した場合、オブジェクトには以下の3つのシンボルが作成され、プログラムからデータにアクセスすることができる。  
- \_binary\_<ファイル名>\_start  
- \_binary\_<ファイル名>\_end  
- \_binary\_<ファイル名>\_size  

例えば画像ファイルをオブジェクトファイルに変換し、プログラム内で上記シンボルからアクセスすることができる。  
これらのシンボルはリンカldのbオプションでバイナリを指定した際にも作成される。  
