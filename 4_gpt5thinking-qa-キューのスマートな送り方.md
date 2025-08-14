ご指摘のとおり、\*\*「APIM（API Gateway）→ Functions（Lambda相当）→ Queue → コンテナ」\*\*は、Functions が“単なる転送”しかしていないなら冗長になり得ます。
ただし、**バリデーション／監査／多重化制御（冪等化・重複検出キー）／テナント境界の強制**をどこで担保するかが肝です。下記の“よりスマート”な代替を要件別に提示します。

---

# ① 標準・実務バランス型（最推奨）

**APIM → 極薄の Ingest API（Functions でも Container Apps でも可）→ Service Bus → Container Apps Jobs（KEDA）**

* Ingest API の責務は最小限：**認可（Entra/JWT）検証・スキーマ検証・サニタイズ・相関ID付与・MessageId/重複検出キー生成・レート制御**のみ。
* **Managed Identity + RBAC** で Service Bus に送信（秘密情報ゼロ）。
* **Queue は Private Endpoint + Private DNS**、Ingest は **VNet 統合**で閉域化。
* メリット：実装管理は薄い関数ファイル1つ程度／可観測性（App Insights）と運用（DLQ/再投入）が素直。
* デメリット：最薄でも“1ホップ”は残る（ただし 100LoC 前後の重さ）。

> 実務ではこの形が最も多く、\*\*“冗長”ではなく“ガバナンスの着地点”\*\*になりやすい構成です。

---

# ② ダイレクト投函型（最小ホップ志向）

**APIM → Service Bus（REST）→ Container Apps Jobs**

* APIM の **Managed Identity** で **Service Bus の AAD 認可**を行い、**REST 直接POST**でキューに投入。
* メリット：関数レイヤーを削減、**ホップ最小**。
* デメリット：APIM ポリシーでの **トークン付与／ヘッダ・ペイロード組み立て／エラー再試行・可観測性が貧弱**になりがち。Schema 進化や検証をポリシーで書くのは保守が重い。
* 適用：**高頻度・低複雑**で、APIM側のポリシー運用に自信があり、\*\*“コードをさらに減らしたい”\*\*強い動機がある場合のみ。

> “できる”が、**長期運用の頑健性では①が勝ち**ます。

---

# ③ オーケストレーション内包型（「キューを見せない」）

**APIM → Durable Functions（Orchestrator）→ Activity Functions / Container Jobs**

* キューは **Durable** が内部で管理（表に出さない）。**再実行・長時間処理・ステート管理**を Durable に委譲。
* メリット：ワークフロー／リトライ／タイムアウト等の**制御ロジックを宣言的に**書ける。
* デメリット：Durable の学習コスト、障害解析の視点が Functions 寄りになる。
* 適用：**人手承認や多段非同期**など、明確なオーケストレーション要件がある案件。

---

# ④（特殊）クライアント直投函

**SPA → Storage Queue（短命SAS）→ Container Apps Jobs**

* バックエンドで\*\*超短命SAS（Add 権限のみ、1–5分、IP/HTTPS制限）\*\*を払い出し、SPA が直接 PUT。
* メリット：最小遅延・最小ホップ。
* デメリット：**Public エンドポイント前提**、監査・バリデーションが分散、**Private Link と両立不可**。
* 適用：**低リスクの軽量ジョブ**・厳密な閉域/ゼロトラスト要件がない場合のみ。
* **Service Bus をブラウザ直叩きは現実的でない**（CORS/SDK/権限の観点で非推奨）。

---

## どれが“スマート”か？—意思決定の指針

* **ゼロトラスト／閉域／運用性重視** → **①**（最薄Ingest + MI + Private Link）
* **ホップ最小・超軽量** → **②**（APIM直投函）※保守性に留意
* **多段・長時間・人手承認** → **③**（Durable）
* **超簡素・低リスクユース** → **④**（Storage Queue + 短命SAS）

---

## 既存チェーンの「冗長さ」を減らす具体策

* Functions を\*\*“極薄 Ingest”\*\*に限定（ビジネスロジックはコンテナへ集約）。
* **重複検出（MessageId）・TTL・サイズ上限**などは Functions 側に共通実装し、**監査の踏み絵**として位置づける。
* **APIM で JWT 検証・レート制御・WAF**、Functions は**最小限**の検証と投函に専念。
* 受け側は \*\*KEDA（Service Bus スケーラ）\*\*で自動スケール、**Private Endpoint**で閉域。
* IaC（Bicep/Terraform）で**一気通貫**にデプロイ、**相関ID**で APIM→Ingest→Queue→Worker をトレース。

---

### 結論

* **“Lambda（Functions）を外すこと自体がスマート”とは限りません。**
* エンタープライズ要件では、\*\*最薄の Ingest を1枚置く①が最も現実的な“スマートさ”\*\*です（ガバナンスと運用性の両立）。
* それでもホップを削りたいなら **②**（APIM→Service Bus 直）ですが、**保守性と可観測性の低下**を受け入れる前提になります。
