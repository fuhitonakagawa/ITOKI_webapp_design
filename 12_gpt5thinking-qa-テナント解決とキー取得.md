結論から申し上げると、\*\*テナント解決＝Functions、APIキー取得＝Container App（ワーカー）\*\*が推奨です。
キーポイントは「**キューに秘密情報を載せない**」「**境界（ingest）で権限・整合性を確定**」「**秘密は実行直前に最小権限で取得**」の3点です。

---

## 推奨アーキテクチャ

```
Frontend ─ Entra（OIDC）
       │  Bearer
       ▼
API Management（JWT検証、X-Tenant-Id上書き、レート制御）
       │  Private
       ▼
Functions（ingest）
  - テナント解決（tid→tenant_id, integration設定）
  - DBからsecret参照名取得（※値は保持しない）
  - バリデーション/スキーマ検証/冪等化キー付与
  - Queue Storageへ投入（secretは載せない）
       │
       ▼
Container App（非同期ワーカー）
  - メッセージ受信→相関ID・冪等化チェック
  - Managed IdentityでKey VaultからAPIキーを取得（Just-in-time）
  - 外部API呼出し／結果保存
```

### キューのメッセージ例（秘密は参照のみ）

```json
{
  "correlation_id": "uuid",
  "tenant_id": "tid-xxxx",
  "integration": "vendorX",
  "secret_ref": { "vault_url": "https://kv.example.vault.azure.net/",
                  "name": "tenant-<tid>-vendorX-api-key",
                  "version": null },  // nullなら最新。厳密再現が必要ならversionを固定で入れる
  "payload": { ... },
  "dedupe_key": "hash-of-business-id",
  "enqueue_ts": "2025-08-14T05:30:00Z"
}
```

---

## なぜこの分担か（理由）

* **秘密の露出最小化**：
  キューは平文ペイロード前提のため、**APIキーを載せない**のが原則。**取得はワーカーで JIT** が安全。
* **ローテーション耐性**：
  キュー滞留中にキーが更新されても、**最新のキーを取得して実行**できます（固定再現が必要なら version をメッセージに入れる）。
* **境界でのガバナンス**：
  Functions 側で **JWT検証結果に基づくテナント解決**・**入力スキーマ検証**・**冪等化キー生成**・**監査メタ付与**を一元化できます。
* **最小権限**：
  Functions の MI は **Queue “send”** 権限のみ、
  Container App の MI は **Key Vault “secrets/get”** と外部API到達のみ――**権限分離**が明確。

---

## 実装上の要点

* **Functions（ingest）**

  * APIMで上書きした `X-Tenant-Id` を受け、\*\*DB（Azure SQL / Cosmos）\*\*で `secret_name`・エンドポイント等の **参照情報のみ取得**。
  * **冪等化**：`dedupe_key` を業務IDから生成、重複投函の抑止（Storage Queueは重複検出がないため自前で）。
  * **セキュリティ**：Functions は **Key Vaultのgetは不要**（付与しない）。Queue は **Private Endpoint**＋**MI: “add” のみ**。

* **Container App（ワーカー）**

  * 受信→`dedupe_key` で **一度きり実行**（Redis/DBで実行記録）。
  * **Key Vault 取得を短時間キャッシュ**（プロセス内LRU/Redis、TTL=数分）。Key更新は **KVイベント→キャッシュ無効化**で対応。
  * 外部API通信は **NAT Gatewayで送信元固定**（先方ホワイトリスト向け）。
  * **可観測性**：`correlation_id` を App Insights に伝播、APIM→Functions→Queue→Worker を貫通。

* **Key Vault/DB 設計**

  * DBには**秘密を置かず**、`secret_name` のみ。
  * Key Vaultは **ソフトデリート＋パージ保護**。ワーカーMIに `secrets/get` 最小付与。
  * テナント厳格分離が必要なら **KV分割（テナント毎）** か **ワーカーMIの分離**を検討。

---

## 例外ルール（この限りでない場合）

* **同期APIで即時外部呼び出し**が必要なエンドポイントのみ：
  Functions で Key Vault 取得→外部API 呼出し。ただし **秘密をレスポンス/キューに載せない**こと。
* **秘密を使用しない委任トークン方式**（外部がOAuth等）なら、Functions で OBO 取得→ワーカーへ参照を渡すなど、要件に応じ最小露出を維持。

---

## チェックリスト

* [ ] キューに**APIキー等の秘密を載せていない**
* [ ] Functions：DB参照は**secret名だけ**／**MIはQueue送信権限のみ**
* [ ] Container App：**MIでKey Vault “get”**／外向きはNAT固定
* [ ] **冪等化キー**・**相関ID**・**スキーマ検証**をingestで実施
* [ ] Keyローテ：**version固定の要否**をユースケース毎に決定（再現性 or 最新性）

---

### まとめ

* **テナント解決は Functions（境界で確定）**、\*\*APIキー取得は Container App（実行直前）\*\*が最も安全で運用性も高い分担です。
* キューには **秘密ではなく“参照（secret\_name/version）”** を載せる方針がベストプラクティスです。
