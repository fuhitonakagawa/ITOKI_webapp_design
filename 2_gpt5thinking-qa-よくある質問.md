以下は、この非同期抽出アーキテクチャで**よく聞かれる質問（FAQ）と推奨回答**のひな型です。レビューや合意形成にそのままお使いください。

# ネットワーク / VPC（Azure では VNet）

* **VPC は使うのか？**
  → \*\*はい。\*\*Azure では *Virtual Network (VNet)* を用います。App Service/Container Apps を VNet 統合し、Storage・Key Vault・Service Bus などは **Private Endpoint（Private Link）** で閉域化します。
* **インターネット入口は？**
  → **Azure Front Door（WAF 有効）** または **Application Gateway (WAF)**。App Service へのオリジン接続は Private Link Origin を推奨。
* **アウトバウンド制御は？**
  → **NAT Gateway** 経由に固定し、必要に応じて **Azure Firewall** で FQDN/タグで許可。
* **DNS は？**
  → Private Endpoint 用に **Private DNS Zone** を作成し、VNet にリンク。
* **リージョン/ゾーン構成は？**
  → 本番は **可用性ゾーン分散**（Front Door はグローバル、下位は同リージョン AZ 分散）。DR が要るなら別リージョンに二重化。

# アイデンティティ / 権限

* **認証は？**
  → **Microsoft Entra ID**（OIDC/OAuth2）。外部ユーザーなら Entra External ID。
* **シークレット管理は？**
  → **Managed Identity + Key Vault** を原則。アプリコードに鍵を置かない。
* **ストレージ権限は？**
  → \*\*Azure RBAC（Storage Blob Data 役割）\*\*を基本。やむを得ない場合のみ短命 **SAS**。

# キュー / 非同期処理

* **どのキューを使う？**
  → 機能要件があるため **Azure Service Bus Queue** 推奨（DLQ、重複検出、スケジュール等）。
  ※**Private Endpoint を必須にする場合は Premium SKU を前提**に検討。
* **ワーカーの実行基盤は？**
  → **Azure Container Apps（Jobs/KEDA）**。Queue 長に応じた自動スケール。簡素化重視なら **Azure Functions（Queue trigger）**。
* **再実行・重複の扱いは？**
  → **少なくとも一度**配信を前提。**冪等化キー**と**重複検出**、**DLQ** 監視を設計。
* **タイムアウトは？**
  → 長時間ジョブはジョブ基盤で実行。フロントは 202 Accepted と **ポーリング/SignalR** で進捗通知。

# ストレージ / データ保護

* **保存先は？**
  → **Azure Blob Storage（ADLS Gen2）**。アカウントは **Public network access: Disabled**、**Private Endpoint** 経由のみ。
* **暗号化は？**
  → 既定の SSE に加え、要件次第で **CMK（Key Vault 管理鍵）**。転送は TLS1.2+。
* **匿名化はどこで？**
  → 図では「システム外」。Azure 内で行うなら **Databricks / Fabric / Synapse** を候補に。

# 通知 / 連携

* **完了通知は？**
  → **Event Grid** → **Functions/Logic Apps** → メール（**Azure Communication Services**）/Teams。UI 即時反映は **Azure SignalR Service**。
* **外部共有は？**
  → 可能な限り RBAC。外部に渡す場合は **短命 SAS（IP 制限・開始/有効期限・権限最小化）**。

# 可観測性 / 運用

* **監視は？**
  → **Azure Monitor + Application Insights + Log Analytics**。相関 ID で分散トレーシング。
* **失敗時の運用は？**
  → DLQ アラート、再投入手順、Poison メッセージ隔離、リトライ/バックオフ方針を Runbook 化。
* **構成ガードレールは？**
  → **Defender for Cloud / Azure Policy** でベースライン強制。

# コスト / スケール

* **コスト最適化の勘所は？**

  * キュー：Service Bus **Standard vs Premium**（Private Link・スループット・冗長性要件で選択）
  * 実行：**Container Apps のジョブ時間 × 台数**が主要コスト。ピークを KEDA で平準化。
  * ストレージ：アクセス階層（Hot/Cool/Archive）とライフサイクル管理。
* **スケール上限は？**
  → 同時実行数・スロットリング（Service Bus の Prefetch/MaxConcurrentCalls）を設計時に上限見積り。

# 信頼性 / 事業継続

* **SLA は？**
  → 各サービスの組合せ。Front Door・Service Bus・Storage・App Service/Container Apps の **個別 SLA** を合成。
* **バックアップ/DR は？**
  → Storage は **GZRS/RA-GZRS**、構成は **IaC（Bicep/Terraform）** で他リージョンに再現。Key Vault の鍵は有効期限・ローテを管理。

# セキュリティ設計の要点（要望されがちな確認）

* **公共ネット遮断**：ストレージ・Key Vault・Service Bus は **Public access 無効**。
* **最小権限**：Managed Identity に最小ロールのみ付与。
* **監査証跡**：診断ログを Log Analytics に集約し、**90日以上保管**＋長期は Storage にアーカイブ。
* **開発フロー**：ブランチ保護＋**CI/CD（GitHub Actions/Azure DevOps）**。承認付きデプロイ、Blue/Green またはスロット。

---

## “要点だけ”の回答テンプレ

* **VNet 使う？** → 使います（Private Endpoint/Link、VNet 統合、NAT、WAF）
* **Queue は？** → Service Bus（DLQ/重複検出）。Private Link要件なら Premium。
* **実行基盤は？** → Container Apps（Jobs+KEDA）。軽量なら Functions。
* **保存先と暗号化は？** → Blob（Private Endpoint、SSE+必要に応じ CMK）。
* **通知は？** → Event Grid → Functions/Logic Apps（メール/Teams/SignalR）。
* **監視は？** → Monitor + App Insights、DLQ アラート、Policy/Defender。

さらに深掘りしたい領域（例：Premium/Standard の選定基準、Front Door vs AppGW、ゾーン/リージョン設計）をお知らせいただければ、設計チェックリストとして落とし込みます。
