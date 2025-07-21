Bonded Validator + DVT

# 概要

Bonded Validator と DVT を利用して拡張可能なステーキング戦略を準備する

それぞれのノードオペレーターとして参入するにあたって、DVT技術を活用・流用する

**ブロックビルダー**もしくは**MEVサーチャー**として活用できるバリデーターを構築運用する

つまり、最終的にはバリデータを運用することでエコシステムの改善とMEV抽出機会を探すことを目的とする

# アドレス設計

ハードウェアウォレットと階層的アドレス構造を踏まえて設計する

この設計は、Trezorの単一マスターシード（リカバリーフレーズ）と、複数のパスフレーズを組み合わせることで、セキュリティレベルと役割に応じてウォレットセットを完全に分離し、各ウォレットセット内でBIP-44アカウントを体系的に利用することを基本方針とする

※ Ledgerを使わない理由は高価なためなので、後でLedgerに変える可能性もあり

## 基本戦略

- **単一の真実の源泉**: 1つのマスターシード（20単語）を物理的に厳重保管（金属板推奨、複数箇所分散）。これが全ての資産への最終的なアクセスキーとなる
    - 後々、マルチシェアバックアップに切り替えられるよう、シングルシェアバックアップとして作成するため、20単語となる ([参考リンク](https://trezor.io/guides/backups-recovery/advanced-wallets/multi-share-backup-on-trezor))
- **パスフレーズによる役割の完全分離**: パスフレーズを変えるだけで全く別のウォレットセット（隠しウォレット）にアクセスできるTrezorの機能を最大限に活用し、セキュリティドメインを分割する
- **BIP-44アカウントによる論理整理**: 各パスフレーズウォレット内で、BIP-44のアカウントインデックス (`account'`) を利用して、プロトコルごと、あるいは目的ごとに資金と権限を論理的に整理する
- **Safe{Wallet}による中央集権的リスクの排除**: 重要な資産管理と操作は、各メンバーが個別に管理するTrezor内のEOAをオーナーとするM-of-Nマルチシグ（Safe{Wallet}）に集約する
- **0xSplitsによる分配の自動化と透明化**: 報酬分配のルールをオンチェーンで管理し、柔軟性と透明性を確保する

## アドレス設計詳細

各役割に対応するEOA（Externally Owned Account）が、どのTrezorウォレットセット（パスフレーズで区別）のどのアカウントから導出されるかを示す

### 凡例:

- **Trezorウォレットセット**: どのパスフレーズで保護されたウォレットかを示す
- **BIP-44パス**: 導出パスの構造 `m / 44' / 60' / account' / change / address_index` を示す
- **EOA**: Externally Owned Account（Trezorで秘密鍵が管理される通常のアドレス）
- **コントラクト**: Safe{Wallet}や0xSplitsなどのスマートコントラクトアドレス

### 階層1: 最重要管理レイヤー（パスフレーズA: MasterAdminVault）

このパスフレーズはチーム内で最も厳重に管理し、アクセスできるメンバーを限定するため、プロトコル全体の根幹に関わる操作に使用する

| 役割                                 | アドレスタイプ | BIP-44アカウント (`account'`) | 導出パスの例 (`m/44'/60'/...`) | 管理・備考   |
| :----------------------------------- | :----------- | :--------------------------- | :---------------------------------- | :--------------------------------------------------------------------------------------------------------------------------------------- |
| **中央報酬管理 Safe{Wallet}** | コントラクト | N/A                          | N/A                                 | 全てのプロトコルからの報酬（ETH, stETH, pufETH, RPLなど）を集約する中央金庫。このコントラクトアドレスが各プロトコルの報酬受取先に指定される。|
| Safe{Wallet} オーナー 1 (EOA)        | EOA          | `Account 0`                  | `0'/0/0`                            | 中央報酬管理Safe{Wallet}のオーナー署名者1。各メンバーが自身のTrezor（`MasterAdminVault`パスフレーズ）のこのパスから導出したEOAをオーナーとして登録。 |
| Safe{Wallet} オーナー 2 (EOA)        | EOA          | `Account 0`                  | `0'/0/1`                            | （オプション）M-of-N構成のための追加オーナー署名者2。 |
| ... (Safe{Wallet} オーナー N)      | EOA          | `Account 0`                  | `0'/0/{N-1}`                        | チームメンバー数に応じて追加。|
| **運用資金管理 Safe{Wallet}** | コントラクト | N/A                          | N/A                                 | （オプション、より高度な分離）ガス代供給、DVT手数料支払い、VT購入などの運用経費を管理するマルチシグ。中央報酬管理Safe{Wallet}から資金を移動して使用。 |
| **0xSplitsマスターコントローラー (EOA)** | EOA          | `Account 1`                  | `1'/0/0`                            | 全ての0xSplitsコントラクトの設定変更権限を持つマスターEOA。このEOA自体をSafe{Wallet}のオーナーとして設定し、0xSplitsの管理もマルチシグで行うことを強く推奨。 |

### 階層2: プロトコル別オペレーションレイヤー（パスフレーズB: ProtocolOperations）

このパスフレーズは、各プロトコルとの直接的な対話（オペレーター登録、ボンド拠出など）を行うEOAを管理する

| 役割                                    | アドレスタイプ | BIP-44アカウント (`account'`) | 導出パスの例 (`m/44'/60'/...`) | 管理・備考 |
| :-------------------------------------- | :----------- | :--------------------------- | :---------------------------------- | :----------------------------------------------------------------------------------------------------------------------------------- |
| **Lido CSM オペレーター管理 (EOA)** | EOA          | `Account 0` (`Lido`)         | `0'/0/0`                            | Lido CSMへのオペレーター登録、ボンド拠出、設定変更など。資金は階層1のSafe{Wallet}から供給。|
| **Rocket Pool ノード管理 (EOA)** | EOA          | `Account 1` (`RocketPool`)   | `1'/0/0`                            | 8ETH Minipoolの作成、RPL担保のステーキング/請求、ノード登録など。報酬請求後、資金は階層1のSafe{Wallet}へ移動。 |
| **Puffer.fi オペレーター管理 (EOA)** | EOA          | `Account 2` (`Puffer`)       | `2'/0/0`                            | Puffer.fiへのNoOp登録、2ETH pufETHボンド拠出、Validator Ticketの購入など。報酬は階層1のSafe{Wallet}へ集約。 |
| **Diva Staking オペレーター管理 (EOA)** | EOA          | `Account 3` (`Diva`)         | `3'/0/0`                            | Diva Stakingへのオペレーター登録、1 divETH担保のロックなど。報酬は階層1のSafe{Wallet}へ集約。 |
| (将来のプロトコルX 管理 EOA)            | EOA          | `Account 4` (`ProtocolX`)    | `4'/0/0`                            | 新たなボンデッドバリデーターやステーキングプロトコルに参加する際の拡張用。アカウントインデックスを増やすことで無限に拡張可能。|

### 階層3: DVTノードキーレイヤー（パスフレーズC: DVTNodeKeys）

このパスフレーズは、オンラインのDVTノードソフトウェアと連携する必要があるオペレーターキー専用で、他の資金や権限から完全に隔離する

| 役割                                       | アドレスタイプ | BIP-44アカウント (`account'`) | 導出パスの例 (`m/44'/60'/...`) | 管理・備考 |
| :----------------------------------------- | :----------- | :--------------------------- | :---------------------------------- | :------------------------------------------------------------------------------------------------------------------- |
| **SSV オペレーターキー 1-1 (EOA)** | EOA          | `Account 0` (`SSV_Cluster1`) | `0'/0/0`                            | SSV Cluster 1のオペレーター1のキー。SSVノードソフトウェアにこのEOAの秘密鍵情報（またはTrezorとの連携）を設定。ガス代は階層1のSafe{Wallet}から少量供給。 |
| ... (SSV オペレーターキー N-M)             | EOA          | `Account N-1` (`SSV_ClusterN`) | `(N-1)'/0/{M-1}`                    | クラスターやオペレーターインスタンスが増えるたびに、アカウントインデックスやアドレスインデックスを増やして対応。 |
| **Obol オペレーターキー 1-1 (EOA)** | EOA          | `Account 10` (`Obol_Cluster1`) | `10'/0/0`                           | Obol Cluster 1のオペレーター1のキー（Charonクライアント用）。SSVとアカウントインデックスを大きく離すことで混同を避ける。|
| ... (Obol オペレーターキー N-M)            | EOA          | `Account 10+N-1` (`Obol_ClusterN`)| `(10+N-1)'/0/{M-1}`                 | 同様に拡張可能。 |

### 階層4: 報酬分配・最終受取レイヤー

| 役割                                      | アドレスタイプ | BIP-44アカウント (`account'`) | 導出パスの例 (`m/44'/60'/...`) | 管理・備考 |
| :---------------------------------------- | :----------- | :--------------------------- | :---------------------------------- | :---------------------------------------------------------------------------------------------------------------------------------------- |
| **0xSplits 報酬分配コントラクト** | コントラクト | N/A                          | N/A                                 | 階層1のSafe{Wallet}から送金された報酬を、設定されたルールに従って分配する。プロトコルごと、あるいは報酬トークンごとに別々のSplitコントラクトを作成することも可能。  |
| **チーム共有経費プール (EOA)** | EOA          | パスフレーズD: `TeamFunds`          | `0'/0/0`                            | 0xSplitsの分配先の一つ。将来の運用コスト、緊急時費用などを積み立てる。このEOAもSafe{Wallet}で管理することでマルチシグ保護下に置ける。  |
| **個人最終報酬受取 EOA (メンバー1)** | EOA          | パスフレーズE: `PersonalPayout1`    | `0'/0/0`                            | 0xSplitsの分配先の一つ。各メンバーが自身で管理する最終的な報酬受取ウォレット。日常的なオペレーションからは完全に分離し、コールドストレージとして扱うことを推奨。このパスフレーズは各メンバーが個人で管理。  |
| ... (個人最終報酬受取 EOA N)                | EOA          | パスフレーズF: `PersonalPayout2`...| `0'/0/0`                            | チームメンバーごとに専用のパスフレーズとウォレットセットを作成。  |

## アドレス設計を実現する際の課題

Trezor SuiteのUIだけでは、私たちが設計したような高度な階層構造を直感的に管理することは困難

「Trezor + MetaMask」という組み合わせを用いることで、Trezorの堅牢なセキュリティを維持したまま、BIP-44の`account'`レベルで完全に分離されたアドレス群を意図通りに生成・管理することが可能になる

これにより、提案したセキュリティと拡張性を両立するアドレス設計を、現実的なオペレーションとして実現することができる

### Trezor SuiteのUI挙動とBIP-44標準の解説

BIP-44で定義されている **「アカウント (`account'`)」と「アドレスインデックス (`address_index`)」** の違いに起因する

- アカウント (`m/44'/c'/X'/...`): 会計上の分離を目的とした、大きな資金のグループです
    - 例えば、「貯蓄用アカウント」「取引手数料用アカウント」のように、目的ごとに完全に分離された勘定と考えることができる
- アドレスインデックス (`.../0/a`): 1つのアカウント内で、トランザクションのプライバシーを高めるために使用される個々のアドレス

Trezor SuiteのUIで「新しいアカウントを追加」をクリックした際に、どの部分がインクリメントされるかについては、歴史的な経緯やUIの設計思想により、ユーザーの観測と技術的な理想が異なる場合がある

- 多くのシンプルなウォレットUIでは、まず`Account 0` (`m/44'/60'/0'/...`) 内のアドレス (`.../0/0`, `.../0/1`, `.../0/2` ...) を順番に生成・表示しようとする
- 多くのシンプルなウォレットUIでは、ユーザーがUI上で「アカウント」と認識しているものが、実際にはこの`Account 0`内のアドレス群である場合がある

Trezorデバイス自体はBIP-44標準に完全準拠しており、任意の導出パス（例えば`m/44'/60'/10'/0/0`など）のアドレスでトランザクションに署名する能力を持っているため、提案したアドレス設計の有効性は変わらない

**問題は「UI上で、どのようにしてそれらのアドレスにアクセスし、管理するか」という点に集約される**

## アドレス設計を実現するための解決策

### 設計を実現するための現実的かつセキュアな解決策

Trezor SuiteのUI上の制約を乗り越え、提案した高度なアドレス設計を実現するための最も現実的でセキュアな方法は、MetaMaskのようなサードパーティ製インターフェースをTrezorと連携させること

#### 具体的な手順（MetaMaskを利用する場合）

1. MetaMaskの準備: ブラウザにMetaMask拡張機能がインストールされていることを確認する
1. Trezorを接続:
    - MetaMaskを開き、右上のアカウントアイコンをクリック
    - 表示されたメニューから「ハードウェアウォレットを接続」を選択
    - Trezorのアイコンを選択し、「続行」をクリック
1. パスフレーズの入力:
    - Trezorデバイスを接続すると、Trezor Connectのポップアップウィンドウが表示される
    - ここで、アクセスしたいウォレットセットに対応するパスフレーズを入力する（例：階層2の「ProtocolOperations」用のアドレスにアクセスしたい場合は、そのパスフレーズを入力し、標準ウォレットの場合は空のまま進む）
1. 【最重要】アカウントの選択:
    - パスフレーズが承認されると、MetaMaskはTrezorから導出可能なEthereumアカウントのリストを表示する
    - このリストには、`m/44'/60'/0'/0/0` (Account 1)、`m/44'/60'/1'/0/0` (Account 2)、`m/44'/60'/2'/0/0` (Account 3) ... というように、BIP-44のaccount'レベルでインクリメントされたアドレスが順番に表示される
    - ここで、設計上使用したいアカウントのチェックボックスを選択します。 例えば、Rocket Pool用に`Account 1` (`1'/0/0`)、Puffer.fi用に`Account 2` (`2'/0/0`) を同時にインポートすることも可能
    - Trezor Suiteでは表示されなかった、残高が0の先のインデックスのアカウントも、この画面で選択・インポートできる
1. アカウントのインポートとラベリング:
    - 「ロック解除」をクリックすると、選択したアカウントがMetaMaskのウォレットリストに追加される
    - MetaMask上で、各アカウントに設計通りの名前（例：「HW - Lido Ops」「HW - RocketPool Ops」）を付けることで、管理が容易になる

### このアプローチのメリットとセキュリティ

- 設計の完全な実現: Trezor SuiteのUI上のGap Limit問題を完全に回避し、BIP-44のaccount'レベルで分離されたアドレスを意図通りに利用できる
- 利便性の向上: 一度MetaMaskにインポートすれば、DeFiプロトコルとの連携やトランザクションの実行がスムーズになる
- セキュリティの維持: この方法でも、秘密鍵やリカバリーシードはTrezorデバイスから一切外に出ず、MetaMaskはあくまでトランザクションを構築し、署名をTrezorに要求するだけの「安全な窓口」として機能する

### 実行時の注意点

- トランザクションに署名する際は、必ずTrezorデバイス本体の画面に表示される送金先アドレスや金額、コントラクトデータが正しいことを最終確認する
- 異なるパスフレーズで保護されたウォレットセット（例：ProtocolOperationsとDVTNodeKeys）のアドレスを同時に管理したい場合は、一度MetaMaskからハードウェアウォレットの接続を解除し、再度接続プロセスを繰り返して別のパスフレーズを入力する必要がある

# システム構成

実行クライアントやコンセンサスクライアント、及びDVTで利用するクライアントの比較と選定

## レポジトリ

| No. | Role | Name | URL | Desc |
|-----|------|------|-----|------|
| 0 | documents | nodoc | - | このページを含むドキュメント群 |
| 1 | Reth,Lighthouse | rethhnode | https://github.com/paradigmxyz/reth/tree/main/etc | rethとlighthouseのdocker関連ファイル群 |
| 2 | Erigon | erigonode | https://github.com/zaitsu/erigonode | erigon3のdocker関連ファイル群 |
| 3 | Nethermind,Nimbus | nnimnode | - | NethermindとNimbusのdocker関連ファイル群 |
| 4 | SSV | ssv-stack | https://github.com/ssvlabs/ssv-stack | ssv公式のスタック |
| 5 | Obol | charon-distributed-validator-node | https://github.com/ObolNetwork/charon-distributed-validator-node | obol公式のスタック |


## 実行環境

| No. | Unit | Place | Price | Role | How | Core/Thread (CPU) | Memory | NVMe SSD | mainnet | hoodi | Desc |
|-----|------|-------|-------|-----|-------------------|--------|----------|---------|-------|------|------|
| 1 | **1** | home | - | EL/CL | DIY |  12 core / 24 thread (AMD Threadripper 2920x) | 32GB | 4TB, 300GB, 30GB | trdiym.michizane.com (reth,lighthouse) | trdiyt.michizane.com (erigon3,caplin) | vc1〜vc5 |
| 2 | **1** | home | - | EL/CL | Shuttle | 8 core / 16 thread (Intel Core i7-11700) | 64GB | 2TB | i7bbm.michizane.com (erigon3,caplin) | i7bbt.michizane.com (reth,lightouse) | |
| 3 | 0 | [xserver](https://vps.xserver.ne.jp/) | 720円 / month (12ヶ月) | dvt | vps | 3 vCPU | 2GB | 50GB | - | - |  |
| 4 | **1** | [shin](https://www.shin-vps.jp/) | 840円 / month (12ヶ月) | dvt | vps | 3 vCPU | 4GB | 50GB | - | - | ssv |
| 5 | 0 | [conoha](https://www.conoha.jp/vps/?) | 641円 / month (12ヶ月) | dvt | vps | 3 vCPU | 2GB | 100GB | - | - |  |
| 6 | 0 | [kagoya](https://www.kagoya.jp/vps/) | 715円 / month | dvt | vps | 2 vCPU | 2GB | 200GB | - | - |  |
| 7 | 0 | [webarena](https://web.arena.ne.jp/indigo/) | 1,630円 / month | dvt | vps | 2 vCPU | 2GB | 200GB | - | - |  |
| 8 | **1** | [xserver](https://vps.xserver.ne.jp/) | 1439円 / month (12ヶ月) | dvt | vps | 4 vCPU | 6GB | 150GB | - | - | ssv |
| 9 | 0 | [shin](https://www.shin-vps.jp/) | 1610円 / month (12ヶ月) | dvt | vps | 4 vCPU | 8GB | 100GB | - | - |  |
| 10 | **1** | [conoha](https://www.conoha.jp/vps/) | 1082円 / month (12ヶ月) | dvt | vps | 4 vCPU | 4GB | 100GB | - | - | ssv |
| 11 | 0 | [kagoya](https://www.kagoya.jp/vps/) | 1617円 / month | dvt | vps | 4 vCPU | 4GB | 600GB | - | - | ssv |
| 12 | **1** | [webarena](https://web.arena.ne.jp/indigo/) | 1630円 / month | dvt | vps | 4 vCPU | 4GB | 80GB | - | - | |
| 13 | 0 | [webarena](https://web.arena.ne.jp/indigo/) | 3410円 / month | liquid staking provider ops | vps | 6 vCPU | 8GB | 320GB | - | - | rokectpool |

※ MEV-BoostリレーはCloudRunを利用する

※ obolと接続するバリデータークライアントをどこに置くか決めかねてるが、パフォーマンス次第ではCloudRunやCaaSでも良い気がする

### Locale settings

localeを`C`にします。

```
$ cat /etc/locale.conf
LANG=C
LC_ALL=C
```

上記設定を加えたら、設定を読み込んでください。

```
source /etc/locale.conf
```

### Time synchronization on Linux

ブロックチェーンには正確な時間管理が必要です。Ubuntuでは、systemd-timesyncdが時間を同期するデフォルトです。 そして、[chrony](https://en.wikipedia.org/wiki/Network_Time_Protocol)は代替手段です。

systemd-timesyncd は 1 つの NTP サーバーをソースとして使用し、chrony は複数のサーバー (通常はプール) を使用します。Ubuntuのデフォルトの出荷は、取得することができます 修正されるまでに 600ms も同期がずれています。私のお勧めは、精度を高めるためにchronyを使用することです。

```
apt update && sudo apt -y install chrony
```

chrony が同期していることを確認します: `chronyc tracking`


### Increase Linux Internet speed with TCP BBR congestion control

https://www.cyberciti.biz/cloud-computing/increase-your-linux-server-internet-speed-with-tcp-bbr-congestion-control/

```
vi /etc/sysctl.d/10-custom-kernel-bbr.conf
net.core.default_qdisc=fq
net.ipv4.tcp_congestion_control=bbr
```

```
sysctl --system
```

### FS

- Ethereumノード（`reth`, `erigon` など）では**大量のランダムI/O・高負荷Journal書き込み**が発生する
- 高性能なNVMe SSDを活かすためには、**適切なファイルシステム選定・フォーマット・マウントオプション**が必要になる

#### 📋 推奨構成

| 項目 | 推奨値 | 備考 |
|-----|-------|------|
| **ファイルシステム** | `ext4` または `xfs`（両方高速、安定） | |
| **フォーマットオプション** | `lazy_itable_init=0`, `-m 0` | あるいは `discard` を追加 |
| **マウントオプション** | `noatime,nodiratime` | あるいは `,discard` を追加 | 
| **パーティション方式** | GPT（推奨） |

#### 🛠 フォーマット手順（ext4）

##### ① デバイス確認

```bash
lsblk -o NAME,SIZE,MOUNTPOINT
```

##### ② GPTでパーティション作成

```bash
sudo parted /dev/nvme0n1 -- mklabel gpt
sudo parted -a optimal /dev/nvme0n1 -- mkpart primary ext4 0% 100%
```

##### ③ ext4 フォーマット

```bash
sudo mkfs.ext4 -F \
  -E lazy_itable_init=0,lazy_journal_init=0 \
  -m 0 /dev/nvme0n1p1
```

###### 各オプションの意味

- `-F`: 強制フォーマット
- `-E`:
    - `lazy_itable_init=0`: inode初期化を即時実行
    - 必要に応じて `discard`: TRIM命令を有効化（SSD向け）
- `-m 0`: ext4の予約領域を0%に（デフォルト5%削除）

---

#### 📦 マウント手順

```bash
sudo mkdir -p /mnt/ethereum
# discard付きの場合
sudo mount -o noatime,nodiratime,discard /dev/nvme0n1p1 /mnt/ethereum
# discard無しの場合
sudo mount -o noatime,nodiratime /dev/nvme0n1p1 /mnt/ethereum
```

##### `/etc/fstab` に永続化する場合

```
/dev/nvme0n1p1 /mnt/ethereum ext4 defaults,noatime,nodiratime,discard 0 2
```

---

#### 🧪 ベンチマーク（オプション）

```bash
sudo apt install fio
fio --name=test --filename=/mnt/ethereum/testfile \
  --size=1G --bs=4k --rw=randrw --ioengine=libaio --iodepth=64 \
  --runtime=60 --time_based --group_reporting
```

---

#### 🔁 代替: `xfs` を使いたい場合

##### フォーマット

```bash
sudo mkfs.xfs -f /dev/nvme0n1p1
```

##### マウント

```bash
sudo mount -o noatime,nodiratime,discard /dev/nvme0n1p1 /mnt/ethereum
```

##### `/etc/fstab` 記述例

```
/dev/nvme0n1p1 /mnt/ethereum xfs defaults,noatime,nodiratime,discard 0 2
```

---

#### ⚠️ 注意点

| 項目 | 内容 |
|------|------|
| `barrier=0` | 書き込み速度は向上するが、**電源断時にデータ破損リスクが高い**（本番環境では非推奨） |
| `discard` | 即時TRIMを行うが、代わりに `fstrim -av` を定期実行する構成も可 |
| `ext4` vs `xfs` | `ext4` は保守性と安定性、`xfs` は高IO性能と大容量向き |

---

#### ✅ まとめ

```bash
# ext4 + SSD向け最適化でフォーマット
sudo mkfs.ext4 -F -E lazy_itable_init=0,lazy_journal_init=0,discard -m 0 /dev/nvme0n1p1

# 推奨マウント
sudo mount -o noatime,nodiratime,discard /dev/nvme0n1p1 /mnt/ethereum
```

この設定により、**Ethereumノードの初回同期（特にrethやerigon）で最大I/O性能**を引き出せます。

### Disk

#### 整理

```
sudo apt autoremove --purge
docker system prune -a --volumes
```

## Execution Client / Consensus Client

実行クライアント、ビーコンクライアントは可能な限り複数の実装から選んで分散化させることが望ましい

一方で幅広く利用するとトラブルシューティングが難しいため、現時点では下記に記載した実装に焦点を当てる

新しくEL/CLクライアントを追加する場合も上記のいずれかから選ぶことにする（多分Erigonの方が楽だと思われる）

### Execution Client

複数の実装が存在するが、比較的新しくリソース消費と比較してパフォーマンスが高いものを選択する

- Reth
- Erigon

#### bootnodes

- https://ethereum.org/en/developers/docs/nodes-and-clients/bootnodes/
- https://github.com/ethereum/go-ethereum/blob/master/params/bootnodes.go#L35-L37

### Consensus Client - Beacon Client

複数の実装が存在するが、コミュニティが活発で、更新頻度の高く、リソース消費の少ないものを選択する

- Lighthouse
- Erigon/Caplin

#### bootnodes

- https://github.com/eth-clients/hoodi/blob/main/metadata/bootstrap_nodes.yaml
- https://github.com/eth-clients/hoodi/blob/main/metadata/bootstrap_nodes.yaml#L8-L13

### Consensus Client - Validator Client

obol collectiveのcharonに接続するバリデータークライアントを選択する

- Lodestar

obol collectiveのドキュメントからテストされていることが確認できたため、比較検討もしていない

## Bonded Validator

### ノードオペレーター視点での比較検討ポイント

- 収益性: コミッション率の具体的な範囲や設定の自由度、プロトコルトークンによる追加インセンティブの有無とその価値、MEV（Maximum Extractable Value）報酬の分配方法、そして自身が拠出する担保（ボンド）から得られる直接的なステーキング報酬（stETHのリベースなど）が重要です。
- 資本要件: バリデーター運用を開始するために最低限必要な自己資金ETHの量、および追加でロックする必要がある担保（ボンド）の種類、量、そしてそれが報酬を生むのかどうかが比較のポイントとなります。
- 運用負荷・技術的要件: ノードのセットアップや日々のメンテナンスの難易度、必要なハードウェアスペック、セキュリティ管理の負担（例えば、Puffer FinanceのSecure-Signerのような技術支援の有無）、DVT（分散バリデーター技術）のサポート状況などが運用負荷を左右します。
- リスク管理: スラッシング（ペナルティによるETH損失）に対するプロトコルレベルでの保護メカニズム（アンチスラッシング技術、保険、担保による補填など）や、スマートコントラクト自体のリスク、ガバナンスリスクをどう評価するかが重要です。
- 参加のしやすさと柔軟性: 誰でも許可なく参加できるか（パーミッションレス度）、複数のバリデーターを効率的に運用できるか、そしてプロトコルからの離脱の容易さも考慮すべき点です。

### 比較表

| 特徴                         | Lido (CSM)                                                                      | Rocket Pool                                                                         | Puffer Finance                                                                     | StakeWise (V3 Vault)                               | Stader Labs (ETHx - 無許可)                               |
| :--------------------------- | :------------------------------------------------------------------------------ | :---------------------------------------------------------------------------------- | :--------------------------------------------------------------------------------- | :------------------------------------------------- | :---------------------------------------------------------- |
| **LST発行トークン** | `stETH`                                                                         | `rETH`                                                                              | `pufETH`                                                                             | `osETH`                                            | `ETHx`                                                      |
| **参加許可** | CSM: よりオープン (パーミッションレス化、一部初期リスト特典あり)                                | パーミッションレス                                                                    | パーミッションレス                                                                     | パーミッションレス                                       | 無許可制あり                                                |
| **最低自己資金ETH (ボンド)** | **1つ目: 2.4 ETH**<br>**2つ目以降: 1.3 ETH**<br>(EAリスト参加者は初回1.5 ETHの場合あり) | 8 ETH または 16 ETH                                                                 | 1 ETH または 2 ETH                                                                   | 1 ETH～ (オペレーター設定)                           | 4 ETH                                                       |
| **追加担保 (ボンド)** | stETH (上記自己資金ETHがボンドとなり、リベース対象)                                     | RPLトークン (運用ETHの10%～、最低0.8 ETH相当～。価格変動リスクあり)                           | なし (プロトコルがリスク管理)                                                          | なし (Vaultの運用成績が信頼性の証)                     | SDトークン (0.4 ETH相当。価格変動リスクあり)                   |
| **オペレーター収益構成** | ① ETHコミッション (Lido DAO設定)<br>② ボンドstETHのリベース報酬<br>③ MEV (Lidoが管理・分配、一部) | ① ETHコミッション (5-20%範囲でオペレーターが設定可)<br>② RPLステーキング報酬<br>③ MEV (Smoothing Pool等で平均化・分配) | ① ETHコミッション (プロトコル設定)<br>② PUFIインセンティブ (将来的に予定)<br>③ MEV (一部オペレーター還元を設計) | ① ETHコミッション (オペレーターが自由に設定)<br>② MEV (オペレーターが構成・享受、一部または全部) | ① ETHコミッション<br>② SDステーキング報酬<br>③ MEV (一部オペレーター還元を設計) |
| **MEV報酬の扱い** | Lido実行クライアントと連携し、MEV-Boost等を利用。プロトコルが管理し、stETH利回りに反映 (一部が間接的にオペレーター報酬に) | Smoothing Pool参加を推奨。参加によりMEV報酬がプール参加者とオペレーター間で平均化・分配される。 | プロトコルレベルでMEV戦略を最適化し、一部をpufETH利回り向上およびオペレーターに還元する設計（詳細は要確認）。 | VaultオペレーターがMEV戦略を自由に構築・実行し、その収益を享受できる（コミッションに反映）。 | プロトコルレベルでMEV戦略を導入し、一部をETHx利回り向上およびオペレーターに還元する設計（詳細は要確認）。 |
| **運用負荷・サポート** | DVT活用による負荷軽減・耐障害性向上を目指す。Lidoコミュニティによるサポート。CSMの具体的なツール提供は発展途上。 | スマートノードスタック（Dockerベースの自動化ツール）提供。非常に活発なDiscordコミュニティによる手厚いサポート。 | Secure-Signer（秘密鍵をオフラインで管理し運用リスク低減）。RAVeによるハードウェア検証。DVTネイティブサポート予定。低ハードウェア要件。 | オペレーターの技術力と責任範囲が大きい。自由度が高い分、自己解決能力が求められる。コミュニティサポートあり。 | Staderが提供するノードセットアップツール。マルチチェーンでの運用ノウハウ。コミュニティサポートあり。 |
| **リスク管理 (オペレーター)** | ボンドstETHが一部損失補填に機能。Lidoの保険基金。DVT導入によるスラッシングリスク分散（予定）。 | RPLボンドがスラッシング発生時の主要な補填リソース。プロトコルの監査は複数回実施。 | アンチスラッシング技術（Secure-Signer、RAVe）がスラッシングリスクを大幅に低減。プロトコルが一部リスクを吸収する設計。 | スラッシング等のリスクは基本的にVaultオペレーターが全責任を負う。透明性が求められる。 | SDトークンボンドがスラッシング補填の一部として機能。プロトコルの監査実施。 |
| **その他特徴 (オペレーター)**| 最大のLST流動性とブランド力。CSMを通じてLidoの分散化に貢献できる。EigenLayer (wstETH) との連携。 | 最も分散化されたノードオペレーターネットワークの一つ。RPLトークンの価格とエコシステムへの依存。EigenLayer (rETH) との連携。 | 非常に低い資本要件。EigenLayerへのネイティブリステーキングサポート。NoOpsバリデーター委託の可能性（ハードウェア不要の場合も）。 | 高いカスタマイズ性と収益機会の追求が可能。独自ブランドの構築。 | 比較的シンプルな参加構造。マルチチェーンでの実績。EigenLayer (ETHx) との連携。 |

注記:

- TVL (Total Value Locked) は常に変動するため、DeFiLlamaなどの分析サイトで最新の値を確認してください。
- CSMやPuffer FinanceのPUFIトークンの詳細など、一部計画段階・開発中の情報も含まれます。公式ドキュメントやアナウンスで最新情報を確認することが重要です。
- 「最低自己資金ETH」は、バリデーターを1つ運用するために必要なオペレーター側のETH拠出額です。プロトコルはこれに加えてプールされたETHを割り当て、32 ETHのバリデーターを起動します。

### 選定

1. Lido CSM
1. Puffer.fi
1. Rocket Pool

#### 選定理由

- 比較的少額で、且つTVLの規模が大きいもの

## DVT

### 比較表

| 特徴                      | SSV.network                                                                 | Obol Network (Charon)                                                             | Diva Staking (DVT要素)                                                               |
| :------------------------ | :-------------------------------------------------------------------------- | :-------------------------------------------------------------------------------- | :----------------------------------------------------------------------------------- |
| **概要・目的** | パーミッションレスなDVTミドルウェア。バリデーターキーを安全に分散し、運用を委任可能にする。ステーカーとオペレーターを繋ぐオープンマーケットプレイス。 | DVTミドルウェア。分散バリデータークラスター（DV Cluster）の構築と運用を容易にし、バリデーターの耐障害性と分散性を向上させる。 | DVTをネイティブに統合したリキッドステーキングプロトコル。P2Pのオペレーターネットワークで分散バリデーションを実現。 |
| **コア技術** | SSV (Secret Shared Validators) プロトコル。鍵を複数のキーシェアに分割し、異なるオペレーターに分散。 | Charonクライアント。Distributed Key Generation (DKG) とSSV技術を組み合わせ、分散バリデータークラスターを運用。 | DVTクライアント「Diva Client」。DKGとMulti-Party Computation (MPC) を利用してキーシェアを生成・管理。 |
| **鍵管理・分散方式** | シャミア秘密分散 (SSS) を利用。バリデーターキーはオフチェーンで分割・再構築され、オペレーターはキーシェアのみを扱う。M-of-N署名スレッショルド（例: 3-of-4）。 | SSSとDKG。DKGにより、信頼できるディーラーなしでキーシェアを生成・配布。M-of-N署名スレッショルド。 | DKGによりキーシェアを生成。キー自体は存在せず、オペレーターはキーシェアを保持して協調署名。M-of-N署名スレッショルド。 |
| **ネットワーク構造** | パーミッションレスなオペレーターのグローバルネットワーク。ステーカー（またはプール）がバリデーターごとにオペレーターを選択し、クラスターを形成。 | オペレーターがCharonクライアントを実行し、分散バリデータークラスター（DV Cluster）を形成。クラスターはプライベートまたはパブリックに構成可能。 | パーミッションレスなノードオペレーターネットワーク。オペレーターはDiva Clientを実行し、プロトコルによってバリデーターに割り当てられる。 |
| **オペレーター報酬** | ステーカーまたはリキッドステーキングプールが、オペレーターのサービスに対して`SSV`トークンで運用料を支払う。料金はオペレーターが設定可能。 | 現在は主にテストネットや初期導入プログラムでのインセンティブが中心。将来的には`OBOL`トークンによるインセンティブや、サービス料モデルを想定。 | ステーキング報酬の一部をコミッションとして受け取る。コミッション率はプロトコルまたはガバナンスにより決定。追加で`DIVA`トークンインセンティブも可能性あり。 |
| **ネイティブトークン役割** | `SSV`: ネットワークアクセス料支払い、オペレーター登録時のステーキング（ボンド）、ガバナンス。オペレーターはSSVをステークして信頼性とキャパシティを示す。 | `OBOL`: ネットワークインセンティブ、ガバナンス、クラスター形成時の調整機能などを計画（詳細は開発段階）。 | `DIVA`: ガバナンス、プロトコルのセキュリティ（スラッシング保険など）、オペレーターへのインセンティブ（予定）。 |
| **参加要件 (オペレーター)** | `SSV`トークンのステーキング（パフォーマンスと連動）。ハードウェアとネットワーク要件を満たすこと。オペレーターとしての登録とプロファイル公開。 | 技術的要件（Charonクライアントの運用知識）。ハードウェアとネットワーク要件。クラスターへの参加または自身でクラスターを立ち上げる。 | Diva Clientを実行するための技術的要件。ハードウェアとネットワーク要件。プロトコルへの登録。特定の担保は不要だが、パフォーマンスが重要。 |
| **運用負荷・ツール** | オペレーターソフトウェア (SSV Node) 提供。広範なドキュメントと活発なコミュニティ。監視ツールやAPI。 | Charonクライアントソフトウェア提供。DV Launchpad（クラスター形成支援ツール）。詳細なドキュメントとコミュニティサポート。 | Diva Clientソフトウェア提供。セットアップや運用のためのガイド。プロトコルによる監視と調整。 |
| **耐障害性・セキュリティ** | スラッシングリスクの大幅な低減（M-of-Nにより単一障害点を排除）。キーの非管理化。オペレーターの地理的・クライアント分散を促進。 | スラッシングリスクの低減。キーの分散管理とDKGによるセキュリティ向上。クラスター内での冗長性と耐障害性。 | スラッシングリスク低減。キーシェアによる分散化。単一オペレーターのダウンタイムや不正行為に対する耐性。 |
| **採用・エコシステム** | Lido (Simple DVT Module, Community Staking Moduleの一部で活用)、StakeWise, Ankr, Stader Labsなど多数のLSTプロトコルや機関投資家が採用・実験。 | Lido (Obol Moduleでのテストと導入検討)、EtherFi, SwellなどのLSTプロトコルやステーキングプロバイダーが採用・実験。コミュニティ主導のクラスターも多数。 | Diva Stakingプロトコルの基盤技術。EigenLayerとの連携も視野。LSTとしての`divETH`の普及と連動。 |
| **パーミッションレス度** | オペレーターとしての登録はパーミッションレス。ステーカーがオペレーターを選択。 | クラスターの作成と参加はパーミッションレス。信頼できる仲間とプライベートクラスターも形成可能。 | ノードオペレーターとしての参加はパーミッションレス。プロトコルがバリデーターを割り当てる。 |
| **その他特徴** | オペレーターのパフォーマンスと料金を可視化するマーケットプレイス。既存バリデーターのDVT化（キーインポート）をサポート。継続的なプロトコルアップグレード。 | DV Launchpadによるテストネット/メインネットでのクラスター形成支援が充実。教育リソースとワークショップが豊富。エンタープライズ向けソリューションも視野。 | LST発行とDVT運用が一体化。流動性とバリデーター運用の分散化を同時に目指す。AVS (Actively Validated Service) としての機能も期待。 |

### 選定

1. ssv.network
    - https://github.com/ssvlabs/gitbook-docs/blob/main/docs/operators/operator-node/maintenance/troubleshooting.md
1. obol collective

### 選定理由

- ドキュメントが充実していて、構築しやすく、比較的リソース消費が少ないもの

## システム構成図

```mermaid
graph TD
    subgraph ServerA [Server A - EL/CL Node 1]
        direction LR
        A_EL[Reth] --> A_CL[Lighthouse-BN]
    end

    subgraph ServerB [Server B - EL/CL Node 2]
        direction LR
        subgraph Erigon3
            B_EL[Erigon] --> B_CL[Caplin]
        end
    end

    %% EL/CL間のP2P接続 (相互)
    A_EL <--> B_EL
    A_CL <--> B_CL

    %% 各ビーコンクライアントは両方の実行クライアントに接続 (冗長性のため)
    A_CL --> A_EL
    A_CL --> B_EL
    B_CL --> B_EL
    B_CL --> A_EL

    subgraph ServerC [DVT Node 1]
        direction LR
        C_SSV[SSV Node 1]
        subgraph Obol Cluster Node C
            C_VC[Lodestar-VC 1] --> C_Charon[Charon Client 1]
        end
    end

    subgraph ServerD [DVT Node 2]
        direction LR
        D_SSV[SSV Node 2]
        subgraph Obol Cluster Node D
            D_VC[Lodestar-VC 2] --> D_Charon[Charon Client 2]
        end
    end

    subgraph ServerE [DVT Node 3]
        direction LR
        E_SSV[SSV Node 3]
        subgraph Obol Cluster Node E
            E_VC[Lodestar-VC 3] --> E_Charon[Charon Client 3]
        end
    end

    subgraph ServerF [DVT Node 4]
        direction LR
        F_SSV[SSV Node 4]
        subgraph Obol Cluster Node F
            F_VC[Lodestar-VC 4] --> F_Charon[Charon Client 4]
        end
    end

    %% DVTノードからビーコンクライアントへの接続
    C_SSV --> A_CL
    C_SSV --> B_CL
    D_SSV --> A_CL
    D_SSV --> B_CL
    E_SSV --> A_CL
    E_SSV --> B_CL
    F_SSV --> A_CL
    F_SSV --> B_CL

    %% Charonは直接Beacon Clientに接続
    C_Charon --> A_CL
    C_Charon --> B_CL
    D_Charon --> A_CL
    D_Charon --> B_CL
    E_Charon --> A_CL
    E_Charon --> B_CL
    F_Charon --> A_CL
    F_Charon --> B_CL

    %% Lodestar-VCはローカルのCharonに接続
    C_VC -.-> C_Charon
    D_VC -.-> D_Charon
    E_VC -.-> E_Charon
    F_VC -.-> F_Charon

    %% インターネットへの接続 (各EL/CLノード)
    A_EL <--> Internet([インターネット])
    B_EL <--> Internet
    A_CL <--> Internet
    B_CL <--> Internet
```