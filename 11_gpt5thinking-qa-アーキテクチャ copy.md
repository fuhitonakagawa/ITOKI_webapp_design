承知しました。以下は、**マルチテナントでテナント別 API キーを安全に保持・利用する**ための、Azure Functions バックエンド前提の標準アーキテクチャです。

# 全体アーキテクチャ（概要）

```
[SPA] ─(OIDC)─> Entra ID
   │ (Bearer)
   ▼
[Azure API Management]
  - JWT検証（tid/roles）
  - X-Tenant-Id 等のヘッダー付与（APIMで上書き）
   │  (Private)
   ▼
[Azure Functions（HTTP Ingest）]
  1) テナント解決（tid → tenant設定）
  2) Secret 参照名の取得（DB）
  3) Key Vault からAPIキー取得（MI/RBAC）
  4) 外部API呼出 or 非同期キュー投入
   │
   ├──(必要に応じ)▶ [Service Bus Queue] → [Container Apps Jobs/KEDA]
   └──▶ レスポンス返却
```

* **認証**：Microsoft Entra ID（マルチテナントアプリ）
* **API ゲートウェイ**：Azure API Management（JWT 検証・レート制御・WAF 前段）
* **アプリ実行**：Azure Functions（HTTP トリガ）
* **設定DB**（テナントメタデータ）：Azure SQL Database もしくは Azure Cosmos DB
* **秘密情報**：Azure Key Vault（テナント別 API キー）
* **（任意）非同期化**：Azure Service Bus + Container Apps Jobs（KEDA）
* **ネットワーク**：VNet 統合＋Private Endpoint（APIM 内部発行・KV・DB・SB を閉域）

---

## データ設計（最小構成）

### 1) 設定DB（例）

* `tenants`

  * `tenant_id`（Entra の `tid` / または自社のテナントID）
  * `display_name`, `status`, `plan`, …
* `tenant_integrations`

  * `tenant_id`
  * `integration_name`（例：`stripe`, `slack`, `vendorX`）
  * `secret_name`（Key Vault の Secret 名；**値は保持しない**）
  * `endpoint`, `quota`, `extra_config_json` …

> ポイント：**DB には秘密そのものを置かず**、「どの Secret 名を使うか」だけを持たせます。
> → キー輪番時は **Secret の“バージョン”のみ更新**し、`secret_name` は固定にできます。

### 2) Key Vault（命名例）

* `tenant-<tid>-<integration>-api-key`

  * 例：`tenant-72d8...-stripe-api-key`
* 運用：**ソフトデリート＋パージ保護**、アクセスは **Managed Identity (Functions)** に `secrets/get` のみ（管理系は別 MI）。

---

## リクエスト〜呼び出しフロー（詳細）

1. **SPA** が Entra でログイン → **Bearer トークン**を取得。
2. **APIM** が `validate-azure-ad-token` で JWT を検証し、`tid`/`oid`/`roles` などを抽出。

   * ユーザー偽装対策として、`X-Tenant-Id` などのユーザー関連ヘッダーは **APIM で一旦削除 → 検証済クレームから再設定**。
3. **Functions (HTTP)** が受信：

   * `X-Tenant-Id`（= tid）で **DB を引き**、`integration_name` に対応する **`secret_name`** を取得。
   * **Key Vault** から `secret_name` の **最新バージョン**を取得（`DefaultAzureCredential` / MI）。
   * 外部 API を **テナント別の API キー**で呼び出し。
   * 重い処理は **Service Bus** に投入し、\*\*Container Apps Jobs（KEDA スケール）\*\*に委譲。
4. **レスポンス**：202 Accepted（非同期）または 200（同期）。相関ID（`x-correlation-id`）は APIM→Functions→Worker で貫通。

---

## コンポーネント選定の指針

* **DB**

  * スキーマが明確／RLS を使いたい → **Azure SQL**（`SESSION_CONTEXT` で `tenant_id` を流す/RLS）
  * 柔軟なスキーマ／高スループット → **Cosmos DB**（`/tenant_id` をパーティションキー）
* **Secret 管理**：**Key Vault 一択**（DB へ平文保存は不可）
* **キャッシュ**：Key Vault 呼び出し削減に **Azure Cache for Redis** を推奨（キー：`<tid>:<integration>`、TTL 〜 数分）。

  * ※キー輪番時は \*\*Event Grid（Key Vault Secret 更新イベント）\*\*でキャッシュ無効化。
* **ネットワーク**：Functions は **VNet 統合**、Key Vault/DB/Service Bus は **Private Endpoint**、外向きは **NAT Gateway** で固定 IP。
* **権限分離**：

  * 実行系 MI … `secrets/get` のみ
  * 管理系 MI … `secrets/set` / `secrets/list`（テナント管理 UI 用）
  * DB も最小権限（読み取り専用/書き込み専用を分離）

---

## セキュリティと運用の要点

* **テナント分離**：

  * 認可は APIM（スコープ/ロール）＋ Functions 側でも **`tid` を必須**として検証。
  * データアクセスは `tid` で必ずフィルタ（SQL なら RLS/Cosmos ならパーティション）。
* **キー輪番**：

  * Secret 名は固定、**バージョン更新**でローテーション。Functions は **バージョン未指定で取得**して最新を取得。
  * 変更イベントで Redis を無効化。
* **観測性**：

  * App Insights の Telemetry に **`tenant_id` を常時付与**。APIM→Functions→Worker を相関 ID でトレース。
* **レート・課金**：

  * APIM で **テナント別レート制御/クオータ**。DB の `plan` と連動。
* **監査**：

  * 誰がいつ Secret を登録/更新したか（管理 API 経由、Key Vault 診断ログも Log Analytics に集約）。

---

## Functions 実装（概略・Python）

```python
# func.azurewebsites.net/api/ingest など
import os, json
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient
# DB は省略（tenant_id + integration -> secret_name を取得する想定）

KV_URL = os.environ["KEY_VAULT_URL"]
cred   = DefaultAzureCredential()
kv     = SecretClient(vault_url=KV_URL, credential=cred)

def main(req):
    # APIMが付与した検証済みヘッダーを使用（偽装対策でAPIMがoverrideしている前提）
    tenant_id = req.headers.get("X-Tenant-Id")
    integ     = req.params.get("integration")  # 例: 'stripe'
    if not tenant_id or not integ:
        return _bad_request()

    # 1) DBで参照名を引く（例: tenant-<tid>-<integ>-api-key）
    secret_name = lookup_secret_name(tenant_id, integ)  # 実装は省略

    # 2) Key Vaultから最新バージョンを取得
    api_key = kv.get_secret(secret_name).value

    # 3) 外部API呼び出し（例）
    resp = call_external_api(api_key, req.get_json())  # 実装は省略

    return _ok(resp)
```

> 実運用では、Key Vault 取得結果を **短時間キャッシュ**し、**タイムアウト/失敗時のフォールバック**、**入力スキーマ検証**、**相関ID** 付与を実装します。

---

## バリエーション

* **App Configuration 併用**：

  * `tenant_integrations.secret_name` や `endpoint` 等の**非機密設定**を App Configuration に置き、Key Vault 参照で連携（**秘密は KV にのみ**）。
* **完全サーバレス志向**：

  * 非同期の本体処理も **Functions (Queue trigger)** に寄せる（Durable Functions で承認フロー等を持たせる）。
* **外部SaaSのIP制限**：

  * Functions のアウトバウンドを **NAT Gateway** に固定し、そのIPを先方にホワイトリスト登録。

---

### まとめ

* **DB には「テナント設定と Secret の参照名」だけ、秘密は Key Vault** に集約。
* **tid を軸に認可・データ分離・課金/レートを統合管理**。
* **APIM（JWT検証＋ヘッダー上書き）→ Functions（MIでKV参照）→ 外部API/Queue**が、堅牢で拡張しやすい定石構成です。






