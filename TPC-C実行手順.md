# CalvinDBでのTPC-C実行手順

## 概要

CalvinDBは、決定論的な手法を活用してアクティブレプリケーションと分散トランザクションの完全なACID準拠を2フェーズコミットなしで保証するスケーラブルなトランザクションデータベースシステムです。

このドキュメントでは、CalvinDBでTPC-Cベンチマークを実行する手順を説明します。

## 前提条件

### 必要なソフトウェア
- Git
- make, g++
- autoconf, libtool, libreadline-dev, libsnappy-dev, pkg-config
- subversion
- unzip, tar

### 外部ライブラリ
以下のライブラリが必要です：
- protobuf
- glog
- zeromq
- gflags

## インストール手順

### 1. 外部ライブラリのインストール

```bash
# 自動インストールスクリプトを実行
./install-ext
```

### 2. 環境変数の設定

`~/.bashrc`にライブラリパスを追加：

```bash
export LD_LIBRARY_PATH=~/CalvinDB/ext/zeromq/src/.libs:~/CalvinDB/ext/protobuf/src/.libs:~/CalvinDB/ext/glog/.libs:~/CalvinDB/ext/gflags/.libs
```

設定を反映：
```bash
source ~/.bashrc
```

### 3. コンパイル

```bash
cd ~/CalvinDB/src
make -j
```

バイナリは `~/CalvinDB/bin/` に生成されます。

## 設定ファイル

### calvin.conf の設定

クラスター内の全マシンを含む設定ファイル（calvin.conf）を作成：

```
0:0:192.168.122.1:10001
1:0:192.168.122.2:10001
2:1:192.168.122.3:10001
3:1:192.168.122.4:10001
4:2:192.168.122.5:10001
5:2:192.168.122.6:10001
```

フォーマット：`マシンID:レプリカID:IPアドレス:ポート`

この例では3つのレプリカがあり、各レプリカに2台のマシンがあります。

## TPC-C実行手順

### 1. クラスターの準備

```bash
cd ~/CalvinDB

# クラスターの状態確認（6レプリカの場合type=1、3レプリカの場合type=0）
bin/scripts/cluster --command="status" --type=1

# 最新コードの取得とコンパイル
bin/scripts/cluster --command="update" --type=1

# 設定ファイルの配布
bin/scripts/cluster --command="put-config" --type=1
```

### 2. TPC-Cベンチマークの実行

#### オリジナルCalvinDB

```bash
# 3レプリカでの実行
bin/scripts/cluster --command="start" --lowlatency=0 --type=0 --experiment=0 --percent_mp=0 --percent_mr=0 --hot_records=10000 --max_batch_size=100

# 6レプリカでの実行
bin/scripts/cluster --command="start" --lowlatency=0 --type=1 --experiment=0 --percent_mp=0 --percent_mr=0 --hot_records=10000 --max_batch_size=100
```

#### 低レイテンシCalvinDB

```bash
# 3レプリカでの実行
bin/scripts/cluster --command="start" --lowlatency=1 --type=0 --experiment=0 --percent_mp=0 --percent_mr=0 --hot_records=10000 --max_batch_size=100

# 6レプリカでの実行
bin/scripts/cluster --command="start" --lowlatency=1 --type=1 --experiment=0 --percent_mp=0 --percent_mr=0 --hot_records=10000 --max_batch_size=100

# 6レプリカ（強可用性）での実行
bin/scripts/cluster --command="start" --lowlatency=1 --type=2 --experiment=0 --percent_mp=0 --percent_mr=0 --hot_records=10000 --max_batch_size=100
```

### 3. プロセスの終了

```bash
# プロセスの終了
bin/scripts/cluster --command="kill" --lowlatency=0 --type=1
```

## パラメータ説明

### 主要パラメータ

| パラメータ | 説明 |
|-----------|------|
| `--experiment=0` | マイクロベンチマーク（TPC-Cワークロードを含む）を実行 |
| `--percent_mp` | マルチパーティショントランザクションの割合 |
| `--percent_mr` | マルチレプリカトランザクションの割合 |
| `--hot_records` | 競合を制御するホットレコード数 |
| `--max_batch_size` | エポックあたりの最大トランザクション数 |
| `--lowlatency` | 0: オリジナルCalvinDB, 1/2: 低レイテンシCalvinDB |
| `--type` | 0: 3レプリカ, 1: 6レプリカ, 2: 6レプリカ（強可用性） |

### TPC-Cトランザクションタイプ

CalvinDBのTPC-C実装は以下のトランザクションタイプをサポートします：

- **TPCCTXN_SP**: シングルパーティショントランザクション
- **TPCCTXN_MP**: マルチパーティショントランザクション
- **TPCCTXN_SRSP**: シングルレプリカ・シングルパーティション
- **TPCCTXN_SRMP**: シングルレプリカ・マルチパーティション
- **TPCCTXN_MRSP**: マルチレプリカ・シングルパーティション
- **TPCCTXN_MRMP**: マルチレプリカ・マルチパーティション

### TPC-Cデータモデル

- **ウェアハウス**: 倉庫データ
- **ディストリクト**: 地区データ（倉庫あたり10地区）
- **カスタマー**: 顧客データ（地区あたり3000顧客）
- **アイテム**: 商品在庫データ（100,000アイテム）

各トランザクションは、設定可能な競合レベルでこれらのデータに対してTPC-C操作をシミュレートします。

## トラブルシューティング

### よくある問題

1. **ライブラリパスエラー**
   - `LD_LIBRARY_PATH`が正しく設定されているか確認
   - `source ~/.bashrc`を実行

2. **コンパイルエラー**
   - 必要な依存関係がすべてインストールされているか確認
   - 外部ライブラリが正しくコンパイルされているか確認

3. **ネットワークエラー**
   - calvin.confの設定が正しいか確認
   - ファイアウォール設定を確認

## 参考情報

- TPC-C実装ファイル: `src/applications/tpcc.cc`, `src/applications/tpcc.h`
- サーバー起動スクリプト: `src/scripts/calvindb_server.cc`
- 設定関連: `calvin.conf`

bashrcはこれ

```bashrc
LD_LIBRARY_PATH=/home/tkt/Documents/GitHub/CalvinDB/ext/zeromq/src/.libs:/home/tkt/Documents/GitHub/CalvinDB/ext/protobuf/src/.libs:/home/tkt/Documents/GitHub/CalvinDB/ext/glog/.libs:/home/tkt/Documents/GitHub/CalvinDB/ext/gflags/.libs ./bin/scripts/calvindb_server
```
