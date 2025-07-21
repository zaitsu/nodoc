# Obol

[コミュニティステーキングモジュール](https://operatorportal.lido.fi/modules/community-staking-module)(CSM)は、[Lidoプロトコル](https://lido.fi/)の最初のパーミッションレスモジュールであり、誰でも通常の(「バニラ」)イーサリアムバリデーターを実行するよりもはるかに高い資本効率でイーサリアムブロックチェーン上でバリデーターの実行を開始できます。

分散型バリデーター([DVT](https://ethereum.org/en/staking/dvt/))は、複数のノードで実行されるイーサリアムバリデーターです。 [Obol Collective](https://obol.org/)は、実行中の分散バリデーターにパーミッションレスなアクセスを提供するツールのセットです。

このチュートリアルでは、デモンストレーション目的でHoodiテストネットを使用しますが、メインネットにも同じ手順を適用できます。

## Downloading the Rocket Pool CLI

```
mkdir -p ~/bin
wget https://github.com/rocket-pool/smartnode/releases/latest/download/rocketpool-cli-linux-amd64 -O ~/bin/rocketpool
chmod +x ~/bin/rocketpool
echo "export PATH=$PATH:$HOME/bin" >> ~/.bashrc
source ~/.bashrc
rocketpool --version
```
### service install

```
$ rocketpool service install

The Rocket Pool 1.16.0 service will be installed.

If you're upgrading, your existing configuration will be backed up and preserved.
All of your previous settings will be migrated automatically.
Are you sure you want to continue? [y/n]
y

/usr/bin/lsb_release
Step 1 of 8: Installing OS dependencies...
[sudo] password for jamie: 
```

### service configuration

```
$ rocketpool service config
```

後はウィザードで以下の設定をする。ウィザードでは方向キー←↑→↓またはtabで照準を移動し、Enterで選択します。1つ前の画面に戻るときはEscを押します。何も保存せずに中断する場合はCtrl+Cです。

- NetworkはEthereum Mainnetを選択する。（Holeskyテストネットの場合はHoleskyを選択する。）
- Client ModeはExternally Managedを選択する。
- Execution Client(実行クライアント)のHTTP URLはlocalhostを使わずにサーバーのローカルIPを指定する。（例：http://192.168.0.44:8545）
- Consesus Client(コンセンサスクライアント)はLighthouseを選択し、HTTP URLはlocalhostを使わずにサーバーのローカルIPを指定する。（例：http://192.168.0.44:5052）
- Graffitiを聞かれたら何でもいいので16文字までの文字列を入力する。Graffitiとは、ブロック生成をした際に記念に文字列をブロックに刻む機能です。自分は特に入力したい文字列がないので空欄にしています。xy座標と色コードを入力することでGraffiti Wallにピクセル単位で絵を描くことも可能です。
- Metricsを有効化する場合はYesを選択する。CPU・メモリ使用量や蓄積された報酬の確認などができます。報酬関係はスマホアプリからも確認できるのでNoでも問題ないです。
- MEV-Boost Modeを聞かれたらExternally Managedを選択する。
- 上記の設定が完了したらSave and Exitを選択します。設定を確認したい場合はReview All Settingsからできます。
- ウィザードでの設定完了後はSmartnode Stackを起動するか聞かれるのでyを入力してください。

### create validator key

```
$ rocketpool wallet init
っっっっっっ