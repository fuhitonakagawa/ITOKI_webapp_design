以下に、**Azure の IaC（Infrastructure as Code）と、AWS の CloudFormation／Terraform／CDK との対応関係**を整理します。結論から言うと、Azureの“純正”は **ARM Templates と Bicep**、運用管理の上モダンなのが **Deployment Stacks**、開発者体験を整える周辺として **Azure Developer CLI（azd）**／**Deployment Environments** が揃っています。

## 対応表（要点）

| 用途                          | AWS 側                       | Azure 側（第一候補）                                                            | 補足                                                                                                       |
| --------------------------- | --------------------------- | ------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------- |
| **宣言的 IaC 本体**              | **CloudFormation**          | **ARM Templates（JSON）** / **Bicep（DSL）**                                 | Bicep は ARM の上位言語で、Bicep → ARM JSON にトランスパイル。Bicep/ARM は公式のデプロイ基盤です。([Microsoft Learn][1])               |
| **スタックのライフサイクル管理**          | **StackSets / Change Sets** | **Deployment Stacks**                                                    | テンプレートで定義した“束”を一単位で管理。不要化時の detach/delete の動作も制御可能。Blueprints は 2026/7/11 に非推奨化予定。([Microsoft Learn][2]) |
| **テンプレート配布（カタログ化）**         | **Service Catalog**         | **Template Specs**（テンプレートの Azure 内保管）＋**Deployment Stacks**              | Blueprints は Template Specs + Deployment Stacks への移行が推奨。([Microsoft Learn][3])                           |
| **開発者向けのテンプレ＋一括プロビジョニング体験** | （なし／Proton等）                | **Azure Developer CLI（azd）**                                             | リポジトリ内の **Bicep or Terraform** を用いて `azd up` で環境構築。テンプレート仕組みも提供。([Microsoft Learn][4])                   |
| **開発者セルフサービス環境の標準化**        | **Service Catalog**         | **Azure Deployment Environments**                                        | IaC テンプレートで標準環境をセルフサーブ提供。([マイクロソフト アジュール][5])                                                            |
| **マルチクラウド IaC**             | **Terraform（HashiCorp）**    | **Terraform（AzureRM/AzAPI プロバイダ）**                                       | 公式ドキュメントあり。Azure でも一級市民。([Microsoft Learn][6])                                                           |
| **コード（プログラミング言語）で IaC**     | **AWS CDK**                 | *（純正同等はなし）*／選択肢：**CDK for Terraform（CDKTF）** or **Pulumi（Azure Native）** | TypeScript/Python 等で Azure リソースを記述可能。([HashiCorp Developer][7], [pulumi][8])                             |

## 補足と現時点の推奨

* **まずは Bicep**：宣言的で型安全、Azure の最新機能追随が最速。内部は ARM に落ちるためサポート面も安心です。([Microsoft Learn][9])
* **運用は Deployment Stacks を併用**：リソース束を“管理対象”として扱え、テンプレから外した資産の扱い方も制御できます。**Blueprints は 2026年7月11日に非推奨**となるため、新規は Stacks へ。([Microsoft Learn][2])
* **チーム体験は azd / Deployment Environments**：テンプレ配布や“`azd up` で一発セットアップ”の開発者体験を整えるのに有効です。([Microsoft Learn][4], [マイクロソフト アジュール][5])
* **マルチクラウド要件なら Terraform** を選択（AzureRM/AzAPI プロバイダ）。**言語で書きたい**場合は **CDKTF** か **Pulumi（Azure Native）** が実用的です。([Microsoft Learn][6], [HashiCorp Developer][7], [pulumi][8])

必要であれば、\*\*「Bicep + Deployment Stacks + Template Specs + azd」\*\*での最小プロジェクト構成（ディレクトリとCI設定）の雛形をご用意します。

[1]: https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/?utm_source=chatgpt.com "ARM template documentation"
[2]: https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/deployment-stacks?utm_source=chatgpt.com "Create and deploy Azure deployment stacks in Bicep"
[3]: https://learn.microsoft.com/en-us/azure/governance/blueprints/overview?utm_source=chatgpt.com "Overview of Azure Blueprints"
[4]: https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/azd-templates?utm_source=chatgpt.com "Azure Developer CLI templates overview"
[5]: https://azure.microsoft.com/en-us/products/deployment-environments?utm_source=chatgpt.com "Azure Deployment Environments"
[6]: https://learn.microsoft.com/en-us/azure/developer/terraform/?utm_source=chatgpt.com "Terraform on Azure documentation"
[7]: https://developer.hashicorp.com/terraform/cdktf/examples-and-guides/examples?utm_source=chatgpt.com "Examples and Guides - CDK for Terraform"
[8]: https://www.pulumi.com/registry/packages/azure-native/api-docs/provider/?utm_source=chatgpt.com "azure-native.Provider | Pulumi Registry"
[9]: https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/?utm_source=chatgpt.com "Bicep documentation"







承知しました。\*\*「GitHub上のTerraformコード → Azureにデプロイ」\*\*までの標準フローを、実運用でよく使う構成に沿って整理します。要は、**GitHub Actions の OIDC で Entra ID に“短期の信頼”を結び、State は Azure Storage に置き、`plan` と `apply` を自動化**する、という流れです。

---

# 全体像（標準パターン）

```
Dev → GitHub PR
      ├─ CI: terraform fmt/validate/tflint
      ├─ terraform init（backend=Azure Blob）
      ├─ terraform plan（成果をPRに出力）
      └─ 承認後 merge
           └─ CD: terraform apply（main など）
                 └─ Azure リソース作成/更新
```

* **認証**：GitHub Actions →（OIDC）→ Microsoft Entra ID（サービスプリンシパル）
* **Terraform Backend（State）**：Azure Storage（Blob）※ロックは Blob の Lease で担保
* **Provider**：`azurerm`（必要に応じて `azapi` を併用）
* **権限**：SP に **サブスクリプション/リソースグループ**へ最小権限（例：Contributor もしくはカスタムロール）＋ **State容器**に「Storage Blob Data Contributor」

---

# 事前ブートストラップ（初回だけ）

1. \*\*サービスプリンシパル（アプリ登録）\*\*を作成
2. **Federated Credentials（OIDC）** を作成（GitHub repo/branch や environment を紐付け）
3. **Terraform State 用の Storage Account / Container** を作成（バージョン管理・ソフトデリート有効）
4. SP に対し、

   * 対象スコープへ RBAC（最小権限）
   * State コンテナへ「Storage Blob Data Contributor」

> これらは手動 or Bicep/スクリプトで一度だけ用意します。

---

# リポジトリ構成（例）

```
infra/
 ├─ main.tf           # ルートモジュール（プロバイダ/バックエンド宣言）
 ├─ providers.tf
 ├─ variables.tf
 ├─ outputs.tf
 ├─ modules/          # 再利用モジュール
 └─ env/
     ├─ dev.tfvars
     ├─ stg.tfvars
     └─ prod.tfvars
.github/
 └─ workflows/
     ├─ tf-plan.yml
     └─ tf-apply.yml
```

`main.tf`（要点のみ）

```hcl
terraform {
  required_version = ">= 1.7.0"
  required_providers {
    azurerm = { source = "hashicorp/azurerm", version = "~> 3.100" }
    azapi   = { source = "azure/azapi",       version = "~> 1.14" }
  }
  backend "azurerm" {}  # ↓ init で実値を渡す
}

provider "azurerm" {
  features {}
}
```

---

# GitHub Actions（OIDC で Azure にログイン）

`tf-plan.yml`（Pull Requestで Plan）

```yaml
name: terraform-plan
on:
  pull_request:
    paths: [ "infra/**" ]
permissions:
  id-token: write     # OIDCで必須
  contents: read
jobs:
  plan:
    runs-on: ubuntu-latest
    defaults: { run: { working-directory: infra } }
    steps:
      - uses: actions/checkout@v4

      - name: Azure login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}       # SPのアプリID
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - uses: hashicorp/setup-terraform@v3

      - name: Terraform Init (Azure backend)
        run: |
          terraform init \
            -backend-config="resource_group_name=rg-tfstate" \
            -backend-config="storage_account_name=sttfstate123" \
            -backend-config="container_name=tfstate" \
            -backend-config="key=${{ github.ref_name }}.tfstate"

      - name: Terraform Validate/Format
        run: |
          terraform fmt -check
          terraform validate

      - name: Terraform Plan
        run: |
          terraform plan -var-file=env/dev.tfvars -out=tfplan.bin
```

`tf-apply.yml`（main への push で Apply）

```yaml
name: terraform-apply
on:
  push:
    branches: [ "main" ]
    paths: [ "infra/**" ]
permissions:
  id-token: write
  contents: read
jobs:
  apply:
    runs-on: ubuntu-latest
    environment: prod     # 環境保護で承認フローを入れられる
    defaults: { run: { working-directory: infra } }
    steps:
      - uses: actions/checkout@v4

      - name: Azure login (OIDC)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - uses: hashicorp/setup-terraform@v3

      - name: Terraform Init
        run: |
          terraform init \
            -backend-config="resource_group_name=rg-tfstate" \
            -backend-config="storage_account_name=sttfstate123" \
            -backend-config="container_name=tfstate" \
            -backend-config="key=prod.tfstate"

      - name: Terraform Apply
        run: |
          terraform apply -auto-approve -var-file=env/prod.tfvars
```

> ポイント
>
> * **長期秘密は不要**（OIDC）。`azure/login` が ARM\_\* 環境変数を設定し、`azurerm` がそれを利用します。
> * **State は Azure Blob**。**ロック**は Blob Lease で自動。**ソフトデリート/バージョニング**有効を推奨。
> * **環境分離**は「別 State ファイル」「別 RG/サブスク」「Workspace」等で。PR は `plan` のみ、本番は `apply` を環境保護で承認制に。

---

# 代替フロー（簡潔に）

* **Terraform Cloud/Enterprise** を使う

  * GitHub はトリガーのみ。実行は TFC 側（State も TFC）。Azure 認証は OIDC or 変数で設定。
* **Azure DevOps Pipelines** を使う

  * `azure/login` と同様に OIDC で Entra 連携。State は同じく Azure Blob。
* **ローカル実行**

  * `az login` → `terraform init/plan/apply`。RBAC と State の権限は同様。

---

# よくある落とし穴（チェックリスト）

* [ ] **Federated Credentials の subject** がワークフローの実態（branch / environment）と一致しているか
* [ ] SP の RBAC が **最小権限**か（過剰な Owner 付与を避ける）
* [ ] **State 用ストレージのライフサイクル**（ソフトデリート・保持期間）と **アクセス制御**（Public 無効）
* [ ] `-backend-config key=` の命名規約（環境ごとに一意）
* [ ] 並行実行時のロック（Blob Lease で担保、無理に `-lock=false` しない）
* [ ] `azurerm` と `azapi` のバージョン固定（再現性）
* [ ] 機密値は **`terraform output -json` から直接ログに出さない**（`sensitive = true`）

---

## まとめ

* GitHub Actions から **OIDC で Entra 認証** → Terraform が **AzureRM プロバイダ**で操作 → **State は Azure Blob**。
* PR で `plan`、承認後 `apply` の二段構えが基本。
* 長期秘密は持たず、**OIDC + RBAC + 最小権限**、State は **バージョン/ソフトデリート**で保護します。

必要であれば、上記をそのまま動かせる **最小サンプルリポ**（ストレージ/RG を作る簡易ブートストラップ付き）もご用意できます。
