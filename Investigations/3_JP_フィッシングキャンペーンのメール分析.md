# フィッシングキャンペーンのメール分析 - 20260714

クライアントが(未開封の)フィッシングメールを受け取り、中身が気になるということで調査を依頼してきた。クライアントはOutlookからダウンロードした.msgファイルを送ってきてくれた。私は提供されたzipファイルをダウンロードし、中の.msgファイルを展開した。メール本文を確認するために.eml形式に変換し、わかりやすいように「phishing」という名前に変更した。会社名と社員名は本文全体を通して伏せてある。

![alt text](images/VeryHao/files.png)

phishing.emlをテキストエディタで開くと、以下の内容が確認できた(全体のスクリーンショットの一部):

![alt text](images/VeryHao/eml-header.png)
![alt text](images/VeryHao/email1.png)
![alt text](images/VeryHao/email2.png)
![alt text](images/VeryHao/email3.png)
![alt text](images/VeryHao/email4.png)

ここには大きなbase64のかたまりに加えて、8ビットエンコードされたセクションもいくつか含まれている。これらは後でデコードするとして、まずはヘッダーを見てみよう。

![alt text](images/VeryHao/subject.png)

件名のbase64をデコードすると次のようになった: 株式会社xxxx<_recipient-name_>
差出人は「Howell Steven」となっており、メールアドレスは: af47ghy899cm[@]hotmail.com
宛先は: <_recipient-name_>@xxxx.com

![alt text](images/VeryHao/replyto.png)

ヘッダーの中には「Reply-To」アドレスとして barringtonbradley55[@]gmail.com が設定されているのも見つけた。<br>
つまり、差出人名・メールアドレス・返信先のいずれもが一致していないということだ。気になったので、その名前のアカウントが実在するか調べてみた——このアカウントは乗っ取られたものか、あるいは完全に架空のものである可能性がある。元の送信元アドレスはおそらく使い捨てのもので、受信者が返信しようとした場合、実際にはこのアクティブなGmailアカウントにルーティングされる仕組みになっていると思われる。

Kaliの仮想マシンを立ち上げ、まず`holehe`を使ってこのアカウントが120以上のサイトのデータベースのどこかに存在するか調べてみた。結果は1件もヒットなし。<br>
次に`h8mail`でこのメールアドレスが過去のデータ漏洩に含まれていないか確認した。こちらも漏洩なし。<br>
最後に、他に何かOSINT情報が拾えないか`theHarvester`も試してみた。これも成果なし。おそらくこれは使い捨てのアカウントである可能性が高い。<br>
乱雑な文字列のhotmailアドレスの方についても同様に調べてみたが、こちらも結果はゼロだった。

![alt text](images/VeryHao/holehe.png)
![alt text](images/VeryHao/h8mail.png)
![alt text](images/VeryHao/harvester.png)

もう一つ気になった点として、「Reply-To」の下にあるAccept-Languageヘッダーが「zh-CN」に設定されていた。これについては後ほど触れるが、頭の片隅に置いておいてほしい。

先ほど気づいたbase64をデコードしてみたところ、大量の埋め込みHTMLマークアップが見つかった:

![alt text](images/VeryHao/htmltag.png)

これはRTF形式のコンテンツで、中にHTMLタグとShift-JISエンコードされたテキストが埋め込まれていることがわかった。読める形式に変換した。先ほどの.emlファイルにもあったように、8ビットテキストとリンクも存在していた:

![alt text](images/VeryHao/8bit.png)

このbase64/RTFの内容は、ここに示されているものと一致する。今回のbase64はMIME構造の一部にすぎないようだが、念のため確認しておいてよかった。デコードすると、メール本文の実際の内容は以下の通りだった:

![alt text](images/VeryHao/email-text.png)

送信者は受信者を騙し、Google StorageのリンクからWPS.zipというファイル名のファイルをダウンロードさせようとしていたようだ。実際にそのリンクにアクセスしてみたところ、すでに削除されていた——通報されて無効化されたものと思われる。

![alt text](images/VeryHao/wps-zip.png)

というわけでファイル自体を直接入手することはできなかった。ただ、関連する情報はいくつか見つかった。似たフォルダ名(veryhao_123866)を持つファイルがすでにX上で話題になっており、それに対応するVirusTotalのエントリも存在していた。唯一の違いはファイル名で、そちらはLOP.rarだった。https://www.virustotal.com/gui/file/8f3f3af66758d7bf3e8fb88af42a16b25b2faafb6d78a79f4f7fe9fd338408c3

![alt text](images/VeryHao/twitter.png)
![alt text](images/VeryHao/virustotal.png)

VirusTotalでもいくつか調べてみた。<br>
WPS.zipファイルのURLパスで検索してみたところ、VirusTotalの判定は「undetected(未検出)」だったが、Relationsタブを見ると、このURLに関連付けられたファイルが2つあることがわかった。<br>
一つは我々のWPS.zip、もう一つはX投稿にあったlop.rarだ。WPS.zipファイルの方は検出スコアが38/64となっていたので、そちらをクリックしてみた。

![alt text](images/VeryHao/veryhao-virustotal.png)

再びRelationsタブを見ると、元のメールにあったURLが表示されており、さらに下にスクロールするとクリックして表示.exeというファイルがBundled Files(同梱ファイル)として表示されていた。これはWin32の実行ファイルだ。

![alt text](images/VeryHao/click.png)

さて、先ほど触れたAccept-Languageヘッダーの「zh-CN」の件に戻ると——これはメール送信に使われたブラウザの主要言語設定が簡体字中国語であったことを示しており、このフィッシングメールが中国発である可能性が高いことを強く示唆している。この見立てを裏付けるもう一つの根拠として、フォルダ名そのもの「veryhao_123688」がある。中国語で「hao」(好)は「良い」を意味するので、このフォルダ名は要するに「とても良い」という意味になる。

調査を行った当初は、このフィッシングキャンペーンについての詳細情報は公開されていなかったが、最近(初回調査から2か月後)になってVirusTotalがこのファイルに関するレポートを公開した。<br>
VirusTotal上でクリックして表示.exeをクリックして詳細を確認してみた。<br>
以下の最後の画像を見るとわかるように、ファイル名からして、このファイルは実在のWindowsプロセス(secinit.exe)を装っている——これはMITRE ATT&CKでいうところの「マスカレーディング(masquerading)」と呼ばれる手法だ。<br>
そしてスクリーンショットの右側には、VirusTotalの「Attack Campaign Targeting Japanese Organizations Using PoisonX Driver(日本の組織を標的としたPoisonXドライバを用いた攻撃キャンペーン)」というタイトルのレポートが表示されている。このレポートをクリックすると攻撃の詳細な説明が得られる——要約すると、被害者に既知の脆弱性を持つドライバをダウンロードさせ、カーネルレベルのアクセス権を得るという手口だ。ただ、ここで特に注目したいのは、Source Region(発信元地域)が中国と記載されている点だ。これは先ほど気づいたAccept-Languageヘッダーの内容とも一致しており、私の推測が正しかったことの裏付けとなった。

![alt text](images/VeryHao/secinit.png)
![alt text](images/VeryHao/report.png)