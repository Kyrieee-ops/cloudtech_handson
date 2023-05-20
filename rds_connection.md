# DBeaverを使用してAmazon RDSへ接続
DBeaverを利用して、プライベートサブネットに配置したAmazon RDSへ接続する方法について解説します。


## 前提
---
本解説では以下の構成で作成済であることを前提とします。<br>
※各リソースの名前については適宜任意の名称に変更可<br>
※本解説ではdevとすることで開発環境で作成することを前提としています


VPC
|項目名|Name/値|
|---|---|
|VPC|dev-vpc|
|VPC CIDR|10.0.0.0.21|

インターネットゲートウェイ
|項目名|Name|
|---|---|
|インターネットゲートウェイ|dev-vpc-igw|
dev-vpcへdev-vpc-igwをアタッチ済みであること

ルートテーブル
|項目名|Name|送信先|ターゲット|
|---|---|---|---|
|ルートテーブル|dev-vpc-rtb-public|0.0.0.0/0, 10.0.0.0/21|dev-vpc-igw|
|ルートテーブル|dev-vpc-rtb-private|10.0.0.0/21|local|


サブネット
|Name|CIDR|Availability Zone|Route Table|
|---|---|---|---|
|dev-vpc-subnet-public1-ap-northeast-1a|10.0.0.0/24|ap-northeast-1a|dev-vpc-rtb-public|
|dev-vpc-subnet-private1-ap-northeast-1a|10.0.2.0/24|ap-northeast-1a|dev-vpc-rtb-private|
|dev-vpc-subnet-public2-ap-northeast-1c|10.0.1.0/24|ap-northeast-1c||
|dev-vpc-subnet-private2-ap-northeast-1c|10.0.3.0/24|ap-northeast-1c||



## DBeaverとは
---
DBeaverとはSQLを扱いためのクライアントツールです。<br>
[公式サイト](https://dbeaver.io/)では以下のように説明しています。

> 開発者、データベース管理者、アナリスト、およびデータベースを操作する必要があるすべての人のための無料のマルチプラットフォームデータベースツール。MySQL、PostgreSQL、SQLite、Oracle、DB2、SQL Server、Sybase、MS Access、Teradata、Firebird、Apache Hive、Phoenix、Prestoなどの一般的なすべてのデータベースをサポートします。

GUIでSQLを操作できることから学習用クライアントとしてだけでなく、現場での実践においても非常に優秀なツールです。

## DBeaverを使用してAmazon RDSへ接続するまでの流れ
---
全体の流れは以下の通りです。
- EC2を作成します
- RDS用のセキュリティグループを作成します
- RDSサブネットグループを作成します
- RDSをPrivateSubnetに作成します
- DBeaverをインストールします
- DBeaverのSSH Tunnel設定します
- DBeaverから踏み台サーバを経由してRDSに接続します


## EC2を作成する
---
1. 名前: step-server1(仮)
2. アプリケーションおよびOSイメージ: Amazon Linux2
3. インスタンスタイプ: t2.micro
4. キーペア: 新しいキーペアを作成: 
    - キーペア名: stepserver-key
5. ネットワーク設定
    - VPC: dev-vpc
    - サブネット: dev-vpc-subnet-public1-ap-norheast-1a
    - パブリックIPの自動割り当て: 有効化
    - セキュリティグループを作成
        - セキュリティグループ名: step-server-sg
        - 説明: step-server-sg
    - インバウンドセキュリティグループのルール
        - タイプ: ssh(デフォルトの設定)
        - ソースタイプ: 任意の場所、ソースに0.0.0.0/0(デフォルト設定)
6. ストレージを設定
デフォルト設定のまま

上記を設定し、インスタンスを起動します。

## Amazon RDSのセキュリティグループを作成
---
RDS用のセキュリティグループを作成します。<br>
EC2のセキュリティグループで作成したstep-server-sgをインバウンドルールに設定することで、step-server-sgの対象のリソースからアクセスを許可することが可能です。

1. セキュリティグループ名: rds-sg
2. 説明: rds-sg
3. VPC: dev-vpc
4. インバウンドルール: 
    - タイプ: MYSQL/Aurora
    - ソース: カスタム, step-server-sg
 5. アウトバウンドルール: 
    - タイプ: すべてのトラフィック
    - 送信先: カスタム, 0.0.0.0/0
6. タグ-オプション: 
    - キー: Name
    - 値: rds-sg

上記を設定し、セキュリティグループを作成します。

## RDSサブネットグループを作成
---

1. サービス名の検索窓から「rds」で検索し、Amazon RDSの画面を開きます。
2. 左ペインのサブネットグループを選択します。
3. DBサブネットグループを作成をクリックします。
4. サブネットグループの詳細: 
    - 名前: dev-subnet-group
    - 説明: dev-subnet-group
    - VPC: dev-vpc
5. サブネットを追加: 
    - アベイラビリティゾーンを選択: 
        - ap-northeast-1a
        - ap-northeast-1c
    - サブネットを選択: 
        - ap-northeast-1aの10.0.2.0/24のsubnetを選択
        - ap-northeast-1cの10.0.3.0/24のsubnetを選択


## Amazon RDSをプライベートサブネットに作成
---
1. Amazon RDS画面の左ペインからデータベースを選択します。
2. データベースの作成をクリックします。
3. データベース作成方法の選択: 標準作成
4. エンジンのオプション: MySQLを選択
5. エンジンバージョン: デフォルトのまま
6. テンプレート: 開発/テスト環境を選択、もしくは料金が気になる方は無料利用枠を選択
7. 設定: 
    - DBインスタンス識別子: 任意の名前を入力(デフォルトのままでも問題ありません)
    - マスターユーザー名: admin
    - マスターパスワード: 任意のパスワードを入力
    - マスターパスワードの確認: マスターパスワードと同様のものを入力
8. インスタンスの設定: 
    - DBインスタンスクラス: バースト可能クラス(t クラスを含む)
    - db.t2.microを選択
9. ストレージ: 
    - ストレージタイプ: 汎用SSD(gp2)
    - ストレージ割り当て: 20GiB
    - 最大ストレージしきい値: 1000Gib
10. 接続: 
    - コンピューティングリソース: EC2コンピューティングリソースに接続しない
    - VPC: dev-vpcを選択
    - DBサブネットグループ: dev-subnet-groupを選択
    - パブリックアクセス: なしを選択
    - VPCセキュリティグループ: 既存の選択
    - 既存のVPCセキュリティグループ: rds-sgを選択
    - アベイラビリティゾーン: ap-northeast-1a
    - 追加設定: データベースポート3306
11. データベース認証: 
    - データベース認証オプション: パスワード認証
12. 追加設定: 
    - データベースの選択肢: 
    - 最初のデータベース名: testdatabase
    - DBパラメータグループ: default.mysql8.0
    - オプショングループ: default.mysql8.0
    - バックアップ: 
        - 自動バックアップを有効にします: チェック
        - バックアップ保持期間: 7日間

上記以外の選択肢はデフォルトのままでとします。<br>
これでデータベースの作成をクリックします。


## DBeaverのインストール
---
1. 以下リンクからDBeaverをダウンロードします。

https://dbeaver.io/download/

2. インストーラを起動しPlease select a language. は日本語を選択しOKをクリックします。
3. DBeaver Communityセットアップへようこそ画面: 次へをクリックします。
4. 使用許諾契約: 同意するをクリックします。
5. Choose Users: For me(user)を選択し、次へをクリックします。
6. 構成要素の選択: インストールする構成要素を選択: デフォルトのまま次へをクリックします。
7. インストール先の選択: 特に指定がなければ次へをクリックします。
8. スタートメニューのフォルダの選択: 特に指定がねｋればインストールをクリックします。
9. DBeaver Community セットアップの完了: 完了をクリックします。


## DBeaverを使用してAmazon RDSへ接続
---
それではいよいよDBeaverからAmazon RDSへ接続します。<br>
DBeaverからAmazon RDSへ接続する際に、

1. DBeaverを起動します。
2. サンプルのデータベースを作成しますか？のメッセージが表示されますが、RDS作成時に最初のデータベース名で入力したデータベースが作成されているため、ここでは作成不要とし、いいえを選択します。
3. 新しい接続タイプを選択する画面でMySQLを選択し、次へをクリックします。
4. 接続先情報を入力します。
    - Server Host: RDSのエンドポイントを入力
    - Datebase: 空白のまま
    - ユーザー名: admin
    - パスワード: RDS作成時に作成したマスターパスワードを入力
5. タブをSSHのタブに切り替えます。
    - 「SSH Tunnel」を使用するのチェックボックスにチェック
    - Host/IP: EC2のパブリックIPアドレス
    - User Name: ec2-user
    - Authentication Method: Public Keyを選択
    - Private Key: EC2作成時にダウンロードしたpemファイルを絶対パスで指定

上記を設定したらテスト接続をクリックします。
初めて対象のサーバに接続する際には、

```
The authenticity of host EC2のパブリックIPアドレス can't be esblished. ssh-ed25519 key fingerprint is xxxxxxxxxxxxxxxx. Are you sure you want to continue connecting?
```
と聞かれることがありますが、「はい」を選択します。
ここでいうサーバとはAmazon RDSのことを指しています。<br>
SSHでサーバに初めて接続した際には、本当にそのサーバに接続して良いですか？という確認のメッセージが表示されます。<br>
fingerprint is xxxxxxxxxxxxxxxx.と表示されているように、これはフィンガープリントと言われる、サーバの指紋です。<br>
SSH接続ではホスト認証というものが行われますが、その際に使用されるのこのフィンガープリントです。<br>
ホスト認証とはSSHでクライアントがサーバに接続する際に、そのサーバが本当に接続したいサーバであるかを確認することです。

接続テストボタンをクリックし、接続済みの画面が表示されれば、問題なくAmazon RDSへの接続が完了です。

