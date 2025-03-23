---
title: "SteampipeとExcel Power Queryで実現するAWS構成定義書自動化ガイド"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS", "Excel", "PowerQuery", "Steampipe", "構成定義書"]
published: true
---

## はじめに

手動による AWS 環境構成定義書の作成は、継続的なメンテナンスが必要であり、大きな負担となります。また、情報の更新漏れや整合性の欠如など、多くの問題が生じます。  

本記事では、Steampipe を用いて AWS リソースの属性情報を CSV 形式で抽出し、その CSV ファイルを Excel の Power Query 機能で読み込むことで、構成定義書の作成プロセスを大幅に自動化します。CSV ファイルの抽出や Excel へのインポートは手動操作が必要ですが、その手順自体はシンプルで再現性があります。

## 構成定義書のサンプル

以下は、本手法で作成できる構成定義書のサンプルです。この例では、EC2 インスタンスの構成情報を取得し、Excel シートに読み込んでいます。

![alt text](/images/spac/image-3.png)

## 本手法のメリット

この手法には以下のメリットがあります。

- single source of truth の実現

  AWS 環境の現在の構成を最新で唯一の正確な情報源（single source of truth: SSOT）とみなします。自動生成された構成定義書は、システムの状態を反映する「読み取り専用の表現」であるため、手動更新によるメンテナンスの必要性がなくなります。システム変更時は、Steampipe で最新情報を取得し、Excel で読み込むだけでシステムとドキュメントの同期を維持できます。

- 構築手法に依存しない

  Steampipe が直接 AWS API から構成情報を取得するため、マネージメントコンソール、CLI、または IaC ツール（例えば、CloudFormation、Terraform、CDK など）などの構築手法に依存しない汎用的な手法です。

- システムのライフサイクルに依存しない

  システムのライフサイクルに依存せず、いつでも最新の情報を取得できます。特に、構成定義書が存在しない、あるいは定義書を更新できずに陳腐化してしまった状況において、システムの実態を把握するために有効です。  

## 構成情報を CSV で抽出する

このセクションでは、Steampipe の概要とセットアップ方法、そして実際に AWS リソース（ここでは EC2 インスタンス）の構成情報を CSV 形式で抽出する手順を説明します。

### Steampipe とは

[Steampipe](https://steampipe.io/)は、SQL でクラウドリソースの情報を抽出できるオープンソースツールです。プラグインを導入することで、AWS、Azure、GCP、Kubernetes など、さまざまなクラウドプロバイダーのリソース情報を SQL で取得できます。Steampipe を利用することで、AWS リソースの属性値を効率よく取得できます。

### AWS CloudShell とは

今回は Steampipe の実行環境として AWS CloudShell を利用します。

AWS CloudShell は、マネージメントコンソールから直接アクセスできる、ブラウザベースのシェル環境です。

https://docs.aws.amazon.com/ja_jp/cloudshell/latest/userguide/welcome.html

CloudShell で利用する credential はマネージメントコンソールから引き継がれるため、IAM ユーザーさえあれば、認証情報のセットアップは不要です。

### Steampipe をセットアップする

マネージメントコンソールにサインインし、ヘッダにある CloudShell のアイコン（`>.`の形をしています）をクリックします。

以下のコマンドを CloudShell のターミナルで実行し、Steampipe と AWS プラグインをインストールします。

```bash
sudo /bin/sh -c "$(curl -fsSL https://steampipe.io/install/steampipe.sh)"
steampipe plugin install aws
```

https://steampipe.io/downloads?install=linux

:::message
Steampipe はホームディレクトリの外部にインストールされるため、CloudShell のセッションが終了すると削除されます。永続化するには、ホームディレクトリに手動でインストールする必要があります。

> Want more control over the install process?
>
> Steampipe is also available as a binary executable (or you can build from source). To manually install Steampipe, unzip the executable and move it to a directory included in your system"s PATH. You can access the source, zipped executables and binary checksums from the releases section of the Steampipe Github repo.
:::

### EC2 インスタンスの構成情報を抽出する

以下の手順で、AWS の EC2 インスタンスの構成情報を CSV 形式で抽出します。

https://hub.steampipe.io/plugins/turbot/aws/tables/aws_ec2_instance

1. SQL クエリをファイル `ec2-ins.sql` に記述する

    ```bash
    cat << EOF > ./ec2-ins.sql
    SELECT
      tags->>'Name' AS "名前",
      instance_id AS "インスタンスID",
      instance_type AS "インスタンスタイプ",
      image_id AS "イメージID",
      private_dns_name AS "プライベートDNS",
      private_ip_address AS "プライベートIP",
      public_dns_name AS "パブリックDNS",
      public_ip_address AS "パブリックIP",
      vpc_id AS "VPC ID",
      subnet_id AS "サブネットID",
      placement_availability_zone AS "アベイラビリティゾーン",
      root_device_name AS "ルートデバイス名",
      key_name AS "キーペア",
      platform_details AS "プラットフォーム",
      architecture AS "アーキテクチャ"
    FROM aws_ec2_instance
    WHERE NOT instance_state = 'terminated'
    ORDER BY "名前";
    EOF
    ```

2. Steampipe を利用して SQL クエリを実行し、結果を CSV ファイルに抽出する

    ```bash
    steampipe query --output csv ./ec2-ins.sql > ./ec2-ins.csv
    ```

    抽出される CSV ファイルのサンプルです。※出力に ID 等が含まれますが、該当環境は検証後に削除済みです

    ```bash
    cat ./ec2-ins.csv 
    名前,インスタンスID,インスタンスタイプ,イメージID,プライベートDNS,プライベートIP,パブリックDNS,パブリックIP,VPC ID,サブネットID,アベイラビリティゾーン,ルートデバイス名,キーペア,プラットフォーム,アーキテクチャ
    ec2Bastion,i-02059f9af149877ba,t2.micro,ami-0599b6e53ca798bb2,ip-172-16-0-217.ap-northeast-1.compute.internal,172.16.0.217,ec2-54-248-206-252.ap-northeast-1.compute.amazonaws.com,54.248.206.252,vpc-0743c3e1c8762fa02,subnet-0596b3ed4c21c6abb,ap-northeast-1a,/dev/xvda,kpair-test,Linux/UNIX,x86_64
    ec2Web,i-0b9aee37d0d95137b,t2.micro,ami-0599b6e53ca798bb2,ip-172-16-2-166.ap-northeast-1.compute.internal,172.16.2.166,,,vpc-0743c3e1c8762fa02,subnet-08517c5867d9a34be,ap-northeast-1a,/dev/xvda,,Linux/UNIX,x86_64
    ```

3. CloudShell のメニューから CSV ファイルをダウンロードする
  
    CloudShell のメニューから[アクション]→[ファイルのダウンロード]を選択し、`ec2-ins.csv` をダウンロードします。

## 構成情報を Excel シートに読み込む

このセクションでは、Power Query の概要と、先のセクションで抽出した CSV ファイルを Excel シートに取り込む手順について説明します。

### Power Queryとは

Power Query は、Excel に組み込まれたデータ取得および変換ツールで、さまざまなデータソースから情報を取り込み、整形できる機能です。これにより、Steampipe で抽出した CSV ファイルのデータを Excel シートに読み込むことが可能となります。

### CSV ファイルを Excel シートに読み込む手順

1. Excel を起動し、[データ]タブを選択する
2. [データの取得]→[テキストまたは CSV から]をクリックし、先ほどダウンロードした`ec2-ins.csv`を選択する
3. [データの変換]をクリックする

    ![alt text](/images/spac/image.png)

4. Power Query エディターが起動したら、[１行目をヘッダーとして使用]をクリックする

    ![alt text](/images/spac/image-1.png)

5. 設定が完了したら、[読み込み]をクリックして、Excel シートにデータをインポートする

    ![alt text](/images/spac/image-2.png)

CSV データを元に Excel のテーブルが作成されます。これが構成定義書となります。

サンプルを再掲します。

![alt text](/images/spac/image-3.png)

## 構成変更を定義書に反映する

Steampipe で取得した構成情報は、最新の情報です。そのため、システムの構成変更があった場合、以下の手順で構成定義書を更新することができます。

検証用の環境に以下の変更を加えます。

- Bastion インスタンスのインスタンスタイプを `t2.micro` から `t3.micro` に変更
- Web2 インスタンスを追加

環境を変更した後で、`EC2 インスタンスの構成情報を抽出する`セクションの手順をもう一度実行し、抽出された CSV ファイルで先に出力したファイルを上書きします。

次に、Excel のテーブルを右クリックし[更新]を実行すると、最新の CSV ファイルが読み込まれ、テーブルが更新されます。

![alt text](/images/spac/image-4.png)

テーブルが更新されると、Excel シートに最新の構成情報が反映されます。

![alt text](/images/spac/image-5.png)

## まとめ

本記事では、Steampipe と Excel Power Query を利用して、AWS のリソース情報から自動的に構成定義書を生成する手法を解説しました。  

このアプローチにより、手動によるメンテナンス負荷を大幅に削減でき、かつ常に最新の構成情報を基にした信頼性の高い構成定義書を維持することが可能になります。

## 次のステップ

今動いているシステムを SSOT とみなし、そのシステムから構成ドキュメントを自動生成した場合、そのドキュメントは、ある関心事に焦点を当てたシステムの状態を写し出す副産物とみなすことができます。

例えば「サーバーへの通信許可」が関心事であるならば、EC2 インスタンスとセキュリティグループの情報を抽出することで、インスタンスに許可された通信を把握できます。このように、システムの構成情報を様々な視点で抽出することで、システムの特定の側面に焦点を当てた構成定義書を生成することが可能です。

次回は SQL JOIN で関連リソースの情報を結合し、特定の関心事に焦点を当てた構成定義書を生成する手法について解説する予定です。
