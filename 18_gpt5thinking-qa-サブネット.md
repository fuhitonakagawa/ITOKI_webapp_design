承知しました。ご提示アーキテクチャに対しての\*\*VNet/サブネット設計の推奨（Azure版VPC）\*\*を、実装に直結する粒度で整理します。結論としては、**Hub-Spoke**を基本にし、**Private Endpoint＋Private DNS**、**固定アウトバウンド（NAT または Firewall）**、\*\*サービスごとの“委任（delegation）サブネット”\*\*を用意する構成が堅牢です。

---

# 1) トポロジ選択

* **推奨**：**Hub–Spoke**

  * **Hub VNet**：共通ネットワーク機能（Azure Firewall / Bastion / Private DNS Resolver など）
  * **Spoke VNet（App）**：アプリ系（APIM、Functions の VNet 統合、Container Apps 環境、Private Endpoint 群 ほか）
  * Hub↔Spoke は **VNet ピアリング**（転送許可）
* 小規模/早期検証なら **単一 VNet** でも可（のちに Hub を増設可能なアドレス設計に）

---

# 2) アドレス計画（例）

* Hub：`10.0.0.0/20`

  * `AzureFirewallSubnet` : `10.0.0.0/26`
  * `AzureBastionSubnet` : `10.0.0.64/27`（任意）
  * `dns-resolver-inbound` : `10.0.0.96/27`（任意）
  * `dns-resolver-outbound`: `10.0.0.128/27`（任意）
* Spoke(App)：`10.0.16.0/20`

  * **`apim-subnet`**（APIM v2 注入用・委任）: `10.0.16.0/24`
  * **`functions-integration`**（Functions の *regional VNet integration* 用・委任）: `10.0.17.0/24`
  * **`aca-infra`**（Container Apps 環境・委任）: `10.0.18.0/23`（スケール余地確保）
  * `private-endpoints`（PE を収容）: `10.0.20.0/24`
  * `shared-services`（Redis/監視エージェント等 任意）: `10.0.21.0/24`

> いずれも余裕を持たせるため /23〜/24 を目安に。後から縮小は困難、拡張は容易です。

---

# 3) サービス配置と“委任”ルール

* **API Management (APIM v2)**：`apim-subnet` に **VNet 注入（Standard v2 / Premium v2）**

  * インバウンドは **Front Door（WAF）** から（外向きPublicでも APIM 側で IP 制限/ヘッダ検証）。Private 化したい場合は **内部モード＋プライベート経路**（AFD Private Link オリジン等）。
* **Azure Functions（HTTP Ingest）**：`functions-integration` に **regional VNet integration**（アウトバウンド私設化）

  * **受け口（HTTP）を私設化**したい場合は **Functions 側に Private Endpoint**を張り、**APIM から私設呼び出し**。
* **Azure Container Apps（非同期ワーカー/KEDA）**：`aca-infra` に **Managed Environment** を作成

  * **固定アウトバウンド IP**が要る場合は `aca-infra` に **NAT Gateway**（もしくは Hub の **Azure Firewall** へ UDR で 0/0 迂回）
* **Private Endpoints**：`private-endpoints` に下記を収容

  * Cosmos DB（`privatelink.documents.azure.com`）
  * Blob／Queue Storage（`privatelink.blob.core…` / `privatelink.queue.core…`）
  * Key Vault（`privatelink.vaultcore.azure.net`）
  * （必要に応じ）Functions（`privatelink.azurewebsites.net`）、APIM、ACR など
* **Static Web Apps / Front Door**：SWA は VNet 内に置けません（PaaS 公開）

  * 厳格に閉域化したい場合は **Storage Static Website or App Service**＋**Private Endpoint**＋**AFD Private Link オリジン**の組み合わせに変更を検討

---

# 4) ルーティング／固定アウトバウンド

* **選択肢 A（簡素）**：各サブネットに **NAT Gateway** を関連付け（`functions-integration` / `aca-infra`）
  → 外部 API への送信元を**サブネットごとに固定**
* **選択肢 B（集中管理）**：Hub に **Azure Firewall**、Spoke 側の UDR で `0.0.0.0/0 → Firewall`
  → **全ワークロードの送信元を一本化**／L3/L4 制御・ログ集中
* どちらでも、**NSG** は最小限許可（443/5671 等）＋管理ポート閉塞

---

# 5) DNS 設計（超重要）

* **Private DNS Zones** を作成し **Spoke（必要なら Hub も）にリンク**

  * `privatelink.blob.core.windows.net`
  * `privatelink.queue.core.windows.net`（Queue Storage を使う場合）**または** `privatelink.servicebus.windows.net`（Service Bus の場合）
  * `privatelink.vaultcore.azure.net`
  * `privatelink.documents.azure.com`（Cosmos）
  * `privatelink.azurewebsites.net`（Functions を私設化する場合）
* **Private DNS Resolver**（Hub）を置くと、オンプレ/DNS 連携やフォワードが楽（任意）

---

# 6) インバウンド経路（整理）

```
[Internet]
  → Front Door (WAF/Rate) ──→ Static Web Apps（公開PaaS）
  → Front Door (WAF/Rate) ──→ APIM（VNet注入/外部公開 or 内部Private）
       └─(Private Endpoint/DNS)→ Functions（HTTP Ingest）
```

* **APIM→Functions** は **Private Endpoint + Private DNS** で呼び出し。
* **APIM→バックエンド Storage/Cosmos/KV** も同様に **Private Endpoint 経由**。

---

# 7) 非同期経路（整理）

```
APIM → Functions（ingest）
    → Queue（Storage Queue or Service Bus）
    → Container Apps（KEDA）
    → Key Vault（MIでget）→ 外部API（NAT or Firewallで固定）
    → Blob（PE）→ Event Grid/Logic Apps → ACS Email
```

* **Queue の Private Endpoint** と **KEDA の名前解決**（Private DNS）が肝
* **Service Bus** を選ぶ場合は Premium で Private Link を前提に検討すると運用が楽

---

# 8) どのサービスが「VNet 外」か

* **VNet 外（直接サブネットは不要）**：Front Door / Static Web Apps / Entra ID / Communication Services / GitHub Actions
* **VNet 内（サブネット必須）**：APIM（v2）/ Functions（VNet Integration 用）/ Container Apps（環境）/ Private Endpoints 群
* **“VNet 外”の PaaS と VNet 内のリソース接続**は **Private Endpoint＋Private DNS** で私設化

---

# 9) 最小セットの“作るべきサブネット”（再掲）

* `apim-subnet`（APIM v2 注入：委任 `Microsoft.ApiManagement/service`）
* `functions-integration`（Functions VNet 統合：委任 `Microsoft.Web/serverFarms`）
* `aca-infra`（Container Apps 環境：委任 `Microsoft.App/managedEnvironments`）
* `private-endpoints`（PE 収容：委任不要／**NSGで受信拒否**＋必要な発信のみ許可）
* （任意）Hub 側 `AzureFirewallSubnet` / `AzureBastionSubnet` / DNS Resolver in/out

---

# 10) 仕上げチェックリスト

* [ ] **Private Endpoint + Private DNS**：Cosmos / Blob / Queue(or Service Bus) / Key Vault / Functions(必要時)
* [ ] **固定アウトバウンド**：NAT GW（サブネット単位） **or** Firewall（Hub 集約）
* [ ] **APIM→Functions** を **私設**で疎通（公衆入口は Front Door のみ）
* [ ] **KEDA の参照先**（Queue）に到達し、名前解決がプライベートを向く
* [ ] **NSG/UDR**：最小許可、既定許可に頼らない
* [ ] **アドレス余裕**：`aca-infra` は /23 以上、将来のスケールを考慮
* [ ] **監視**：Log Analytics / App Insights を VNet 経由（必要なら PE）で集約

---

## まとめ

* **Hub–Spoke**でネットワーク機能を集約し、**委任サブネット（APIM/Functions/ACA）＋Private Endpoint サブネット**を基軸に構成します。
* **Private Endpoint＋Private DNS**、**固定アウトバウンド（NAT / Firewall）**、**NSG/UDR の最小許可**を押さえれば、現在のアーキテクチャに綺麗にフィットする VNet 設計になります。
* Static Web Apps だけは VNet 外（公開）なので、**Front Door での入口一元化とヘッダー/IP 制御**でガードしてください。






いいえ、**サブネットに入るのは Container Apps だけではありません。**
ご提案アーキテクチャに沿うと、VNet 内には次の“置き場所”が必要になります。

---

## 1) サブネットに「入る／紐づく」もの

* **API Management v2（Standard v2 / Premium v2）**
  専用の *委任サブネット* に **VNet 注入**して配置（例：`apim-subnet`）。
* **Azure Functions（HTTP Ingest）の *VNet 統合***
  **アウトバウンド専用**の *統合サブネット* を使用（例：`functions-integration`）。
  ※Functions 本体は PaaSで VNet の外ですが、“統合”のためにサブネットが要ります。
* **Azure Container Apps（Managed Environment）**
  **インフラサブネット**（例：`aca-infra`）が必須。
  ※専用ワークロードプロファイルを使う場合は **ワークロード用サブネット**も追加。
* **Private Endpoints（Private Link の NIC）**
  Cosmos DB / Blob（Queue）/ Key Vault /（必要なら）Functions/ACR 等の **PE を収容する専用サブネット**（例：`private-endpoints`）。
* **固定アウトバウンド経路**
  サブネットに **NAT Gateway を関連付け**（Container Apps / Functions 統合サブネット等）。
  もしくは Hub に **Azure Firewall** を置き、**UDR** で 0/0 を迂回。
* **（Hub 側の共通基盤・任意）**
  **Azure Firewall** 用 `AzureFirewallSubnet`、**Bastion** 用 `AzureBastionSubnet`、
  **Private DNS Resolver** 用 in/out サブネット、（使うなら）**Application Gateway** 用専用サブネット。

---

## 2) サブネット“外”だが Private Endpoint を使うもの

* **Cosmos DB / Blob Storage（Queue Storage）/ Key Vault /（必要に応じ Service Bus, ACR）**
  これらは PaaS 本体は VNet 外ですが、**PE を `private-endpoints` サブネット**に作り、**Private DNS** と組み合わせて私設接続します。
* **Functions のプライベート受け口**（必要時）
  Functions に **Private Endpoint** を張れば、**APIM→Functions を私設**にできます。

---

## 3) 完全に VNet 外（サブネット不要）

* **Front Door / Static Web Apps / Microsoft Entra ID / Communication Services / Event Grid / GitHub / ARM**
  いずれもグローバル PaaS。VNet とは **Private Endpoint（必要なもののみ）** や WAF/Origin 設定で連携します。

---

## 4) 参考：最小のサブネットセット（例）

* `apim-subnet`（**委任**：`Microsoft.ApiManagement/service`）
* `functions-integration`（**委任**：`Microsoft.Web/serverFarms`）
* `aca-infra`（**委任**：`Microsoft.App/managedEnvironments`）
* `private-endpoints`（PE 収容。**委任なし**／NSG・UDR 適用）
* （任意）Hub の `AzureFirewallSubnet` / `AzureBastionSubnet` / DNS Resolver in/out

> 併せて **Private DNS Zones**（`privatelink.blob.core.windows.net`、`…queue…`/`…servicebus…`、`…vaultcore…`、`…documents…`、`…azurewebsites.net` 等）を作成し、VNet にリンクしてください。

---

### まとめ

* **Container Apps だけでなく**、APIM の注入、Functions の VNet 統合、各種 **Private Endpoint** を収容するために **複数のサブネット**が必要です。
* これに **NAT（または Firewall+UDR）** と **Private DNS** を組み合わせるのが、今回のアーキテクチャに対する定石のネットワーク設計です。







いい質問です。**Queue Storage は“サブネットに配置”する対象ではありません**（PaaSのため）。VNet 側には **Private Endpoint（PE）の NIC** を置いて私設接続します。要点だけ整理します。

---

## 結論（ネットワーク）

* **Queue Storage 本体**：VNet 外（PaaS）。
* **VNet 側に置くもの**：`private-endpoints` サブネット内の **Private Endpoint（subresource: `queue`）**。
* **名前解決**：**Private DNS Zone `privatelink.queue.core.windows.net`** を作成し、VNet にリンク。
* **公開遮断**：ストレージ口座は **Public network access: Disabled**（または Selected networks）。
* **疎通要件**：Container Apps / Functions から **443/TLS** で PE のプライベートIPへ到達できること（NSG/UDR で許可）。

```
[Container Apps / Functions サブネット] ──(443, Private DNS)──▶ [PE in private-endpoints Subnet] ─▶ Queue Storage(PaaS)
```

---

## 権限（ID ベース推奨）

* **Managed Identity + RBAC** を基本にし、接続文字列/SASは極力使わない。

  * 送信のみ：`Storage Queue Data Message Sender`
  * 受信/削除：`Storage Queue Data Contributor`（または Reader + Delete 相当）
* Functions（ingest）：**Sender 権限のみ**
* ワーカー（Container App）：**Contributor 相当**（読み取り/削除/可視化タイムアウト変更などが必要なら）

---

## KEDA（Container Apps の自動スケール）での注意

* **スケールルール**：`azure-queue` を使用。

  * 認証は **MI（推奨）** または **接続文字列（シークレット）**。
* **名前解決**：前述の **Private DNS** が効いていないと、KEDA がキュー長を取得できません。
* **ネットワーク**：Container Apps の環境サブネットから **PE へ 443** 到達可能に。
* **ポイズン/重複**：Storage Queue には DLQ/重複検出がないため、**DequeueCount によるポイズン搬送・冪等化キー**を自前で実装。

---

## 設計チェックリスト

* [ ] `private-endpoints` サブネットに **Storage Account（queue）用 PE** を作成
* [ ] **Private DNS Zone** `privatelink.queue.core.windows.net` を作成・VNet にリンク
* [ ] ストレージ口座は **Public network access: Disabled**（可能なら）
* [ ] **Functions MI**：`Storage Queue Data Message Sender`
* [ ] **Container App MI**：`Storage Queue Data Contributor`
* [ ] **KEDA**：`azure-queue` ルール＋（MI または接続文字列）で監視
* [ ] NSG/UDR：アプリ側サブネット → PE へ **443** 許可、不要な東西/南北は閉塞
* [ ] 運用：`nslookup <account>.queue.core.windows.net` が **PE のプライベートIP**を返すことを確認

---

### ひと言まとめ

> **Queue Storage 本体はサブネット外**。VNet には **Private Endpoint** を置き、**Private DNS** でキューの FQDN をプライベートIPへ向けます。
> 権限は **Managed Identity + RBAC**、スケーリングは \*\*KEDA（azure-queue）\*\*で。










拝見しました。結論から言うと、**サブネットに入れるのは Container Apps だけではありません**。この構成だと、**APIM（v2 の VNet 注入）／Functions の VNet 統合用サブネット／Container Apps 用サブネット／Private Endpoint 用サブネット**が必要です。逆に **Cosmos／Blob／Queue／Key Vault／ACR などの PaaS 本体は VNet 外**で、**Private Endpoint（PE）** をその専用サブネットに作って“私設接続”します。

---

# 推奨レイアウト（例）

VNet: `10.10.0.0/16`

* **`apim-subnet`**（委任: `Microsoft.ApiManagement/service`）: `10.10.1.0/24`

  * APIM v2 を **VNet 注入**。Front Door/WAF 経由で到達させ、APIM→下流は私設通信。
* **`functions-integration`**（委任: `Microsoft.Web/serverFarms`）: `10.10.2.0/24`

  * Functions の **regional VNet integration**（アウトバウンド私設化）。
  * 受け口も私設にしたい場合は **Functions に Private Endpoint** を追加（下の PE サブネットに作成）。
* **`aca-infra`**（委任: `Microsoft.App/managedEnvironments`）: `10.10.3.0/23`

  * Azure Container Apps の **Managed Environment**。KEDA から Queue へ到達できること（後述の DNS/PE 前提）。
* **`private-endpoints`**（PE 収容）: `10.10.10.0/24`

  * **Cosmos / Blob / Queue Storage / Key Vault / ACR / Functions(必要時)** それぞれの **PE** をここに配置。
  * NSG は**最小許可**（アプリ側サブネット → 443/TCP のみなど）。
* （任意・Hub）`AzureFirewallSubnet` / `AzureBastionSubnet` / DNS Resolver in/out

> CIDR は余裕を持って /24 以上、Container Apps は将来スケールを見込み **/23** を推奨。

---

# Private DNS（必須）

PE を使う先の **Private DNS Zone** を作り、VNet にリンクします。

* `privatelink.blob.core.windows.net`（Blob）
* `privatelink.queue.core.windows.net`（Queue Storage ※将来 Service Bus に替えるなら `privatelink.servicebus.windows.net`）
* `privatelink.vaultcore.azure.net`（Key Vault）
* `privatelink.documents.azure.com`（Cosmos DB）
* `privatelink.azurecr.io`（ACR）
* `privatelink.azurewebsites.net`（Functions を PE で私設化する場合）

> これが無いと、KEDA/アプリが**パブリック FQDN に解決**してしまい、Private Link を使えません。

---

# アウトバウンド固定（どちらか）

* **簡素:** `functions-integration` と `aca-infra` に **NAT Gateway** を関連付け（送信元グローバル IP を固定）。
* **集中管理:** Hub に **Azure Firewall** を置き、Spoke サブネットに **UDR(0/0→Firewall)**。運用ガバナンスが強い場合に有効。

---

# 各サービスの置き場所・接続まとめ

* **Static Web Apps / Front Door / Entra / Communication Service / GitHub / ARM** … VNet 外（サブネット不要）
* **API Management v2** … `apim-subnet`（VNet 注入）
* **Functions** … 本体は PaaS（VNet 外）だが **`functions-integration` に統合**。受け口を私設化したいなら **PE を `private-endpoints` に作成**して APIM→PE 経由で呼ぶ。
* **Container Apps** … `aca-infra`（必要に応じ workload サブネット追加）
* **Cosmos / Blob / Queue / Key Vault / ACR** … それぞれ **PE を `private-endpoints`** に作成し、**Private DNS** で私設解決。
* **Queue Storage** … 本体は VNet 外。**PE（subresource: `queue`）**＋`privatelink.queue.core.windows.net`。
* **KEDA（Container Apps スケール）** … 上の Private DNS が効いていないと**キュー長を取得できない**ので要確認。

---

# よくあるハマりどころ（チェック）

* [ ] **PE は作ったが Private DNS を作っていない／VNet にリンクしていない**
* [ ] **APIM→Functions を公衆経路で呼んでいる**（PE＋Private DNS で私設化）
* [ ] **NAT/Firewall 未設定で外部APIの送信元IPが変動**（ホワイトリストに弾かれる）
* [ ] **Queue Storage の DLQ/重複検出を前提にしている**（機能が無いので自前実装 or **Service Bus** 検討）
* [ ] **MI + RBAC 未整備**（Functions: Queue *Sender*、Container Apps: Queue *Contributor*、KV: *secrets/get* のみ、ACR: *AcrPull*）

---

## まとめ

* **サブネットに入れるのは**：APIM(VNet注入)／Functions(統合)／Container Apps／Private Endpoints。
* **PaaS 本体（Cosmos/Blob/Queue/KV/ACR）は VNet 外**で、**PE＋Private DNS** により私設接続。
* **NAT or Firewall** で外向きを固定し、NSG/UDR は最小許可。
  この方針で配置すれば、図のアーキテクチャに対して **破綻のないネットワーク／サブネット設計**になります。必要でしたら、上記の CIDR をそのまま使える **Bicep/TF の雛形**もお出しできます。
