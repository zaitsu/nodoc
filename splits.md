# 🖥 0. 前提確認

- macOS ターミナルが使えること
- Git, curl がインストールされていること（通常プリインストール）

## 🧭 Step 1: Foundry をインストール

```
# Foundry で必要らしい
brew install libusb

# Foundry の公式インストーラーを取得
curl -L https://foundry.paradigm.xyz | bash

# ターミナルを再読み込み
source ~/.bashrc  # bash
# または
source ~/.zshenv   # zsh

# Foundry ツール群（forge, cast, anvil 等）をインストール
foundryup
```
- これで forge, cast, anvil が使えるようになります

## 🔐 Step 2: テスト専用ウォレットを作成（秘密鍵生成）

```
# ランダム mnemonic から秘密鍵を生成
cast wallet new

# toshza@Ishango work % cast wallet new
# Successfully created new keypair.
# Address:     0x1165dd29A6398dC3A673D9C6526430C9f09aa5E3
# Private key: 0x8cd969118251155683ee3483dc15c2bc59e284a3f5073f2500a3e010572dadbc

# または特定の mnemonic から秘密鍵生成
cast wallet new-mnemonic
```
- 実行後、出力されるのが `ADDRESS` と `KEY`（秘密鍵）
- `KEY` は以降必要です

## 💾 Step 3: `.env` ファイルに設定

プロジェクトルートに `.env` を作成：

```
echo 'PRIVATE_KEY=<ここに cast wallet new の KEY を貼る>' > .env
echo 'RPC_URL=https://rpc.hoodi.ethpandaops.io' >> .env
```
- テストネット目的なので `sushiWallet` や本番キーは使用しないでください

## 📁 Step 4: Split コントラクトリポジトリをクローン

```
git clone https://github.com/0xSplits/splits-contracts.git
cd splits-contracts
forge install

forge install rari-capital/solmate
echo "@rari-capital/solmate=lib/solmate" > remappings.txt

forge build
```

## 🧾 Step 5: デプロイスクリプトの準備

`script/DeployImmutableSplit.s.sol` に以下内容を作成してください：

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.18;

import "forge-std/Script.sol";
import "../src/SplitMain.sol";

contract DeployImmutableSplit is Script {
    function run() external returns (address splitAddress) {
        vm.startBroadcast();

        address;
        recipients[0] = 0xAAA...111;
        recipients[1] = 0xBBB...222;

        uint32;
        allocations[0] = 2500; // 25%
        allocations[1] = 7500; // 75%

        address controller = address(0);

        SplitMain factory = SplitMain(0xSPLIT_MAIN_ADDRESS);
        splitAddress = factory.createSplit(recipients, allocations, 0, controller);

        console.log("✅ Split deployed at:", splitAddress);
        vm.stopBroadcast();
    }
}
```
- `0xSPLIT_MAIN_ADDRESS` は対象ネットワークの `SplitMain` ファクトリアドレスに置き換える

## 🔧 Step 6: デプロイ実行

```
# 環境変数読み込み
source .env

forge script script/DeployImmutableSplit.s.sol:DeployImmutableSplit \
  --rpc-url $RPC_URL \
  --private-key $PRIVATE_KEY \
  --broadcast
```

実行後、Terminal に以下の出力が表示されます：

```yaml
✅ Split deployed at: 0xYourSplitAddress
```

## 🔍 Step 7: 成功確認

1. トランザクションID を確認
    - 出力ログや RPC が返す txhash をメモ
1. Hoodi スキャンで確認
    - https://hoodi.etherscan.io/tx/<txhash> をブラウザで開く
1. Split コントラクトの内容確認
    - 0xYourSplitAddress をスキャンに貼り付け、
    - recipients() や totalRecipients() 関数で設定が正しいかチェック

## ✅ Step 8: テスト送金＆分配確認（任意）

```
# テスト用ETHをSplitに送信
cast send 0xYourSplitAddress \
  --rpc-url $RPC_URL \
  --private-key $PRIVATE_KEY \
  --value 10000000000000000  # 0.01 ETH

# 各受取人の残高確認
cast call <recipient address> balanceOf # ETH用
```
- 正しく受け取れていれば、分配比率通りに送られています
