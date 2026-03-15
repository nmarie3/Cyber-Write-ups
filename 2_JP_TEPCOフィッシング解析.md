# TEPCOフィッシング解析 - 20260315

### AIで翻訳されています。

今回取り上げるのは、東京電力（TEPCO）を騙ったフィッシング詐欺で使われたファイルだ。攻撃者は「toshiba-co.jp」を模した偽メールアドレスを使い、ファイル共有サービス「files.fm」に置いた悪意あるファイルのダウンロードリンクを送りつけるという、よくある手口のフィッシング攻撃を仕掛けてきた。実際に手元に届いたので、さっそく中身を見ていこう。

RARファイルを解凍すると、中に怪しい .js ファイルが入っていた。

![alt text](images/TEPCO/file.png)
![alt text](images/TEPCO/extracted.png)

これはひどい。beautifier.io のような jsビューティファイアにかけて、まず読めるようにしよう。

整形後のコードは2,100行以上。ざっと見た感じ、文字列配列の処理と、中央にどでかい実行関数がひとつある構成だ。つまりこのコードは、配列のインデックスを次々に参照しながら文字列を取り出し、何らかのコマンドを組み立てるだけのものだ。<br>
全部掘り下げるのは時間の無駄なので、読める部分だけを拾っていく。

ActiveXObject が出てきた時点で、これはブラウザ上で動くJavaScriptではないとわかる。Windowsでダブルクリックするだけで実行される、JScriptだ。

![alt text](images/TEPCO/activexobject.png)

おや。外部スクリプトを呼び出すIPアドレスが見つかった。隠す気もなかったのか。Tempフォルダに保存されるようになっている。<br>
http[:]//91.92.243.254:7777/91.92.243.254/vickytwo/ENCRYPTED[.]ps1

![alt text](images/TEPCO/iptops1.png)

"shell" と "http" も確認できる。その上の部分は、文字列を連結して ActiveXObject を生成しているようだ。

![alt text](images/TEPCO/shellhttpactivex.png)

PowerShell が実行されている箇所はこちら。

![alt text](images/TEPCO/executepowershell.png)

下記の2つの文字列配列を見ると、ダミー文字列の合間に、コマンドの断片が混じり込んでいるのがわかる。見つかった部分をすべてハイライトしてある。断片的ではあるが、このコードが何をしようとしているか、少しずつ読み解いていけそうだ。

![alt text](images/TEPCO/fillertextQ.png)
![alt text](images/TEPCO/fillertextm.png)

ハイライトした読み取れる文字列からPowerShellコマンドを組み立ててみよう。<br>
'powershel' + 'l.exe%20-no' + 'p%20-ep%20byp' = **powershell.exe -nop -ep byp**<br>
最後の "ass" の部分が足りない。コード内を検索するとすぐに見つかるはずだ。

![alt text](images/TEPCO/filesearch.png)

ビンゴ！<br>
**powershell.exe -nop -ep bypass -file**<br>
実行ポリシーのバイパスが行われていることが確認できた。<br>
さらに後ろに \x20\x22 が連結されている。ファイルパスか……？
正直なところ、ここまで分解してもまだ核心には届いていない。<br>
残念ながら、このコード単体でわかることはここまでだ。<br>
ただ、ENCRYPTED.ps1 がダウンロード・実行されると、Tempフォルダに何かが展開され、先ほど組み立てたPowerShellスクリプトで実行される流れになっているのはほぼ間違いないだろう。

実際に動かしてみるために、AnyRunで解析してみよう。<br>
執筆時点では、ファイルを配信していたサーバーはすでにダウンしている。幸い、まだ稼働していた頃に別の誰かが同じファイルをアップロードしてくれていた。以降はこのサンドボックスの結果を参照する：<br>
https://app.any.run/tasks/d75c093a-dff4-4652-9a31-490c649a1003

複数の検知が出ているが、まずここに注目しよう。<br>
WinRARの時点ですでにフラグが立っている。実行されると wscript が起動する——これは先ほどの文字列配列にも WScript が含まれていたので、つじつまが合う。<br>
次にPowerShellスクリプトが動く。さっき組み立てたものと同じで、ファイルパスとファイル名が追加されている。場所は確かにTempフォルダ内だ。<br>
続いて aspnet_compiler.exe が登場する——これは後で詳しく見る。<br>
さらにもう一度 wscript.exe が動き、今度は別のファイルを実行するPowerShellスクリプトが走る。これは多段階のドロッパーチェーンと見て間違いないだろう。

![alt text](images/TEPCO/anyrun.png)

では、aspnetファイルの中身を見てみよう。

![alt text](images/TEPCO/aspnet.png)

aspnet_compiler.exe はPC上でWebアプリをビルドするためのツールだが、いわゆる「LOLBin（環境寄生型ツール）」に分類される。Microsoftの正規 .NETフレームワークに署名されているため、セキュリティ製品に検知されにくく、攻撃者に悪用されやすい。このコンパイラが実際に悪意あるコードをビルドし、正規プロセスとして紛れ込むことで検知を回避していると考えられる。

結果を一目見ただけでも、ファイルやWebブラウザからデータを盗もうとしていることがわかる。警告セクションに「Discord/Telegram APIの利用の可能性あり」という記載があるのも興味深い。これらのアプリがC2サーバーとして使われているのかもしれない。
次に、2つのwscriptを見ていこう。

![alt text](images/TEPCO/wscript1.png)
![alt text](images/TEPCO/wscript2.png)

まずコマンドラインを確認する：<br>
C:\Windows\System32\WScript.exe" "C:\Users\admin\AppData\Local\Temp\Rar$DIa10032.37332\TEPCO_CCPP-26Q7305A-N23A.01-DETAILED-RQMT-RFQ.js<br>
これで ENCRYPTED.ps1 が何をしていたか、だいたい見えてきた。Tempフォルダ内に Rar$DIa10032.37332 というフォルダを作成し、その中に元のファイルと同じ名前の TEPCO .js ファイルを生成したのだ。ファイル名を同じにしているのは、見かけ上は同一ファイルに見せかけるためだろう。ただ中身のスクリプトは別物のはずだ。このファイルが実行されると、HTTPリクエストを送信して H41MOD92.ps1 をTempフォルダにダウンロードし、PowerShellでそれを実行する。<br>
このスクリプトが aspnet_compiler.exe を呼び出す役割を担っていると思われる——「アセンブリを動的にロードする」という動作からも読み取れる。

![alt text](images/TEPCO/powershell1.png)

2つ目の wscript.exe も基本的な流れは同じだ。<br>
ただしコマンドラインには別のフォルダ名 Rar$DIa10032.38073 が現れる。新たにフォルダが作られた痕跡は見当たらないため、この2つのフォルダと .js ファイルは、ENCRYPTED.ps1 が実行された時点でまとめて生成されていたと考えられる。つまりそれぞれのスクリプトの内容も異なるはずだ。<br>
このファイルもHTTPリクエストを発行し、4L6MK5IT.ps1 をTempフォルダに落としてPowerShellで実行する。PowerShell のタブを見ると、データをエンコードする処理が行われているようだ。

![alt text](images/TEPCO/powershell2.png)

ただし、最後のPowerShellコマンドの末尾に conhost.exe のエラーコードが残っている。実行されるはずだったスクリプトが途中で失敗し、暗号化されたデータがC2サーバーに送信されなかった可能性がある。

一連の感染の流れを整理するとこうなる：<br>
TEPCO .jsファイル<br>
&nbsp;&nbsp;└── ENCRYPTED.ps1 をダウンロード・実行<br>
&nbsp;&nbsp;&nbsp;&nbsp;└── %TEMP% 内に RarDIa10032.37332とRarDIa10032.37332 と Rar
DIa10032.37332とRarDIa10032.38073 を作成<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;└──  Rar$DIa10032.37332 の TEPCO.jsファイルを実行<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;└── H41MOD92.ps1 を実行<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;└── エラー<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;└── aspnet_compiler を呼び出し<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;└── ペイロードをコンパイル<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;└── Rar$DIa10032.38073 の TEPCO.jsファイルを実行<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;└── 4L6MK5IT.ps1 を実行<br>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;└── エラー

とはいえ、これはあくまで私の見立てに過ぎない。

aspnet_compiler が Snake YARA としてフラグされていたので、このマルウェアがどのようなものか調べてみよう。

オーストラリアの cyber.gov.au によると：<br>
「Snake YARA」ルールとは、Snakeマルウェアを検知するためのセキュリティ検出シグネチャだ。Snakeはロシア連邦保安局（FSB）が開発した高度なサイバースパイツールで、20年以上にわたって世界中の政府機関や重要インフラからの情報窃取に使われてきた。YARAルールは、このマルウェアのファイルパターンやメモリ上の特徴を検出するために設計されている。

……なかなかどうして、面白い相手じゃないか。