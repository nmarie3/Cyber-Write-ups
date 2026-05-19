# QRadar Community Edition インストール手順 - 20260412

QRadar Community Editionのセットアップはちょっと面倒で、完了まで2時間くらいかかります。<br>
この手順はホストPCの仮想マシン上にセットアップすることを前提にしています。<br>
では始めましょう。<br>
まずIBMアカウントとOracle VirtualBoxのインストールが必要です。

IBMアカウントができたら、https://www.ibm.com/community/101/qradar/ce/ にアクセスしてファイルをダウンロードしてください。<br>
各ファイルの役割はこちら：<br>
1. .iso -- QRadarのOSとソフトウェアが入ったインストールメディア。VMにQRadar CEを入れるためのメインファイルです。<br>
2. .sha256 -- ISOのチェックサムファイル。ダウンロードが正常に完了して破損していないか確認するためのものです。<br>
3. .iso.sig -- IBMによる電子署名。ISOが本物で正規のものであることを証明します。<br>
4. .key -- 3ヶ月有効のCEライセンスファイル。3ヶ月ごとに再ダウンロードして更新します。<br>
.keyファイルはとくに重要なので、後で詳しく説明します。

![alt text](<images/install/downloads.png>)

VirtualBoxを開いて「新規」アイコンをクリックします。ここでISOを追加します。<br>
VMに名前をつけましょう。自分はQRadarにしました（後でQRadar CEに変更しました）。<br>
その下はVMの保存先です。デフォルトのままでも、自分で指定してもOKです。<br>
次はISOイメージ。IBMからダウンロードした.isoファイルを探して選択します。<br>
「次へ」をクリックしてください（UIが違う場合は、次のカテゴリのドロップダウンをクリック）。

![alt text](<images/install/VB1.png>)

ユーザー名とパスワードの設定画面が出ます。好きなものを設定してください。<br>
そして「次へ」。

![alt text](<images/install/VB2.png>)

ここは重要です。QRadarはちゃんと動かすためにかなり具体的な要件があります。<br>
RAM/メモリは24 GB、CPUは6コア、ディスク容量は250 GBを推奨します。<br>
4コアでもいけるかもしれませんが、6コアにできるなら6コアにしましょう。<br>
設定が終わったら「完了」をクリックするとVMが自動で起動します。

![alt text](<images/install/VB3.png>)
![alt text](<images/install/VB4.png>)

ただ、起動せずにアボートする場合があります。自分の環境ではあるファイルが既に存在するというエラーでアボートしました。<br>
解決策として、VMの保存先フォルダに生成された謎のファイル4つを削除しました。<br>
何のためのファイルなのか正直よくわかりませんが、削除して再起動したら問題なく動きました。<br>
削除後にDVDを再マウントするよう言われるので、.isoファイルをもう一度追加してください。

![alt text](<images/install/aborted.png>)
![alt text](<images/install/upload-iso.png>)

一つ重要なことがあります！<br>
QRadar CEのVMを起動する前に、設定からアダプターを変更しておきましょう。<br>
ホストマシンからのみ使うならNATでも問題ないですが...<br>
このラボの目的はネットワーク上の別デバイス（自分の場合は自宅サーバー）から実際のログデータを送ることなので、ブリッジアダプターにする必要があります。<br>
デフォルトではNATになっているので、今のうちに変更しておきましょう。<br>
VMを右クリック → 設定 → ネットワークが見つかるまでスクロール、で変更できます。

![alt text](<images/install/bridged.png>)

VMを起動しましょう。<br>
最初にログイン画面が出ます。ユーザー名は`root`、パスワードは空白です。<br>
その後、インストールが始まります。完了まで約1時間かかります。

![alt text](<images/install/hostlogin.png>)
![alt text](<images/install/installing.png>)

初期インストールが終わると、めちゃくちゃ長い使用許諾契約（EULA）が表示されます。<br>
`q`を押してスキップし、`yes`と入力して同意してください。

![alt text](<images/install/bootTOS.png>)
![alt text](<images/install/bootTOS2.png>)

もう少しインストールが続きます。5〜10分くらいです。<br>
QRadar CEのRed Hatブート画面が出たら折り返し地点です。<br>
最初の項目をEnterで選択してください。

![alt text](<images/install/bootscreen.png>)

次の画面はインストール内容の選択です。<br>
「Software Install（ソフトウェアインストール）」をスペースキーで選択してください。<br>
次は「All-In-One Console（オールインワン）」のまま。「Normal Setup（通常セットアップ）」もそのまま。<br>
タイムサーバーの設定画面が出ます。デフォルトのままでもいいですし、`pool.ntp.org`にしてもOKです。<br>
pool.ntp.orgは無料で使える公開NTPプールです。

![alt text](<images/install/setup1.png>)
![alt text](<images/install/setup2.png>)
![alt text](<images/install/setup3.png>)
![alt text](<images/install/setup4.png>)

その後はロケーション・タイムゾーン・IPv4・enp0s3の設定が続くので順番に進めてください

![alt text](<images/install/setup5.png>)
![alt text](<images/install/setup6.png>)
![alt text](<images/install/setup7.png>)
![alt text](<images/install/setup8.png>)

次の画面ではホスト名、IPアドレス、ネットワークマスク、ゲートウェイ、プライマリDNS、セカンダリDNS、パブリックIPを入力します。<br>
ホスト名は適当に決めてください。自分のと似たような感じで大丈夫です。<br>
ホストマシンに戻ってコマンドプロンプトを開き、`ipconfig`を実行してください。<br>
現在のIP・サブネット・デフォルトゲートウェイが確認できます。<br>

![alt text](<images/install/ipconfig.png>)

QRadarサーバーにホストと全く同じIPは使えないので、末尾の数字を255未満の別の値に変えてください。<br>
サブネット（ネットワークマスク）とゲートウェイはそのままコピーでOKです。<br>
重要なのは、ゲートウェイとIPアドレスの最初の3つのブロック（オクテット）が一致している必要があるという点です。一致していないとエラーになります。<br>
プライマリDNSは8.8.8.8（GoogleのDNS）を使うと名前解決が安定します。<br>
8.8.4.4はGoogleのセカンダリ公開DNSです。<br>
パブリックIPは先ほど入力したIPアドレスと同じ値を入れてください。

![alt text](<images/install/setup9.png>)

次はadminパスワードの設定。Webコンソールのログインに使います。<br>
ただ、初回サインイン時にリセットを求められる場合もあるのでそんなに気にしなくて大丈夫です。<br>
その後、新しいrootパスワードも設定します。こちらはVMのターミナル用のパスワードです。

![alt text](<images/install/setup10.png>)
![alt text](<images/install/setup11.png>)
![alt text](<images/install/installdone.png>)

全部終わったらインストール完了です！お疲れさまでした！<br>
早速Webコンソールが動いているか確認しましょう。<br>
ブラウザを開いて`https://<your-qradar-ip>/console`にアクセスしてください。your-qradar-ipはセットアップで設定したIPに置き換えてください。<br>
IPを忘れた場合は、VMのターミナルで`ip a`を実行して確認できます（VirtualBoxならenp0s3を探してその下のinetを見てください）。

![alt text](<images/install/ipa.png>)

うまくいけばこんな感じのUIが表示されます。<br>
ログイン名は「admin」、パスワードは先ほど設定したものです。<br>
パスワードを忘れてしまった場合は、VMのターミナルで以下を実行するとリセットできます：<br>
`/opt/qradar/bin/qradar_password.sh` を実行すると、新しいパスワードの入力と確認が求められます。

![alt text](<images/install/qradar-login.png>)

これでQRadar CEのダッシュボードが見えるはずです！<br>
やったね！

![alt text](<images/install/dashboard.png>)

最後に一つだけ。ライセンスキーについて触れておきます。<br>
初回セットアップでは特に気にしなくて大丈夫ですが、Webコンソールにログインするとライセンスの有効期限に関する警告が表示されます。<br>
以下はその対処方法です。

![alt text](<images/install/update.png>)

ダウンロードページに4つのファイルがあったのを覚えていますか？その中にライセンスキー（.keyファイル）がありましたね。<br>
QRadar CEは無償で使い続けられますが、IBMは3ヶ月ごとにキーを更新します。つまり3ヶ月（ダウンロードのタイミングによってはそれより短い）が過ぎると、QRadarがログの記録を止めてしまいます。使い続けるには新しいキーをダウンロードしてサーバーに適用する必要があります。<br>
面倒に聞こえるかもしれませんが、実は結構簡単です。

VMのターミナルを直接操作する代わりに、SSHで接続して適用することもできます。<br>
ホストマシンから①のターミナルを開いて`ssh root@<qradar-ip-address>`を実行してください。qradar-ip-addressは設定したIPに置き換えてください。<br>
次に②の別のターミナルを開いて以下を貼り付けます：<br>
`scp <path-to-qradar-key>/<qradarkey.key> root@<qradar-ip-address>:/tmp/`<br>
<>の部分は適切な情報に置き換えてください。<br>
ファイルパスをターミナルにドラッグ＆ドロップすると自動で入力されるので便利です。

![alt text](<images/install/adding-key.png>)

scpがやっているのは、ホスト側のファイルをQRadar VMの/tmpフォルダにコピーすることです。<br>
SCP（Secure Copy Protocol）はSSH経由でネットワーク越しにファイルを安全に転送するプロトコルです。<br>
scp [コピー元] [コピー先]という構文で使います。

ファイルがちゃんと転送されたか確認するには、SSHセッションのターミナルで以下を実行してください：<br>
`ls /tmp/*.key`<br>
/tmp内の.keyファイルが一覧表示されます。

![alt text](<images/install/check-key.png>)

確認できたら、SSHターミナルから以下のコマンドで新しいキーを適用します：<br>
`/opt/qradar/bin/qradar_license_key_upload.sh /tmp/<new-.key-file>`

これでまた3ヶ月使えます！<br>
Webコンソールからもライセンスの確認ができます。<br>
ダッシュボードの左側にあるハンバーガーメニューからAdminパネルを開いて、「System and License Management（システムおよびライセンス管理）」をクリックするとライセンス一覧が表示されます。

![alt text](<images/install/sidebar.png>)
![alt text](<images/install/sysconfig.png>)
![alt text](<images/install/licencelist.png>)

もしVM起動時に.isoファイルへのアクセスに関する警告が表示された場合（VMは普通に起動します）、VM設定の「ストレージ」を開いてQRadarの.isoエントリに表示されている黄色い三角アイコンを確認し、そのファイルを削除して「OK」をクリックしてください。<br>
次回起動時から警告は表示されなくなります。

![alt text](<images/install/warning.png>)
![alt text](<images/install/750-qradar.png>)

### インストール完了！