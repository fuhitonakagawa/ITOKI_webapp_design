フォーマルに要点だけまとめます。

# 方針

**プライベートサブネット内のコンテナからキューを取得する方法は2通り**あります。推奨は①です。

## ① Private Endpoint（Private Link）で“私設”接続（推奨）

* 対象：**Azure Service Bus（Queue）** または **Azure Storage Queues**
* 手順（概略）

  1. **VNet 内**に対象リソースの**Private Endpoint**を作成。

     * Service Bus の場合：`<namespace>.servicebus.windows.net`
     * Storage Queue の場合：`<account>.queue.core.windows.net`
  2. **Private DNS Zone**（例：`privatelink.servicebus.windows.net` / `privatelink.queue.core.windows.net`）を作成し、**コンテナの属する VNet にリンク**。
     → FQDN がプライベート IP に解決されます。
  3. 対象リソース側は**Public network access を無効**に（もしくは“選択したネットワークのみ”）。
  4. コンテナは通常の接続文字列/FQDN でアクセス（IP を直書きしない）。資格情報は**Managed Identity＋RBAC**（例：Service Bus Data Receiver/Sender）を推奨。SAS は最小限。
* ポイント

  * 完全閉域（インターネット不要）。
  * **KEDA/Functions のトリガ**も同じ解決結果を使うため、**環境（Container Apps 環境や Functions の VNet 統合）からも Private DNS が参照できること**が必要。
  * 既製/社内 DNS を使う場合は**条件転送**で Private DNS Zone を解決可能に。

> 注：Private Endpoint 利用可の SKU を選択してください（Service Bus は Basic 以外が対象）。

## ② NAT Gateway 経由で“公開エンドポイント”へ出る（Private Link 不使用時）

* コンテナサブネットに **NAT Gateway** をアタッチし、**固定グローバル IP** で外向き通信。
* Service Bus / Storage 側は**ファイアウォールで送信元 IP（NAT のパブリック IP）だけ許可**。
* DNS は通常のパブリック解決のまま。
* メリット：構成が簡素。
* デメリット：インターネット経由（ゼロトラスト要件が厳しい場合は不向き）。

# 設計チェックリスト

* [ ] コンテナ実行基盤（例：**Azure Container Apps（Jobs）/ Functions**）は **VNet 統合**済みか。
* [ ] **Private Endpoint** を配置するサブネットと **Private DNS Zone** の**リンク**は正しいか。
* [ ] **Name Resolution**：`nslookup <namespace>.servicebus.windows.net` が**プライベート IP**を返すか。
* [ ] **ポート許可**：AMQP/TLS **5671** または HTTPS **443** がサブネットの NSG/UdR で許可されているか。
* [ ] **権限**：Managed Identity に `Azure Service Bus Data Receiver/Sender` など最小ロール付与。
* [ ] **スケール連動**（KEDA）：スケーラが同一解決経路で Service Bus に到達できるか（環境のインフラサブネットも Private DNS 参照可に）。

# まとめ

* **閉域・ゼロシークレット重視 → ① Private Endpoint 一択**。
* **簡素・低コストで十分 → ② NAT + IP 制限**。

要件（ゼロトラスト/運用制約/コスト）に合わせて上記いずれかを選べば、プライベートサブネット内のコンテナでも安全にキュー取得が可能です。
