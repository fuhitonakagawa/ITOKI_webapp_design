結論：**どちらでも実現可能**ですが、**リポジトリが GitHub にあり、既に Actions＋OIDC で Azure にデプロイする想定なら、Azure Pipelinesは必須ではありません**。多くのケースで **GitHub Actions 中心**がシンプルで運用しやすいです。
一方で、**組織要件次第で Azure Pipelines を選ぶ理由**もあります。以下に意思決定の基準を整理します。

# クイック判定（どっちを選ぶ？）

* **コードホストが GitHub** → **Actions 推奨**（PR連携・CodeOwners・Checks が自然に使える）
* **Azure DevOps（Boards/Repos/Test Plans）を中核運用** → **Pipelines** が一体運用で楽
* **厳格な変更管理・承認を Azure DevOps 側で統一** → **Pipelines**
* **オンプレ/閉域（Azure DevOps Server）で CI/CD を完結したい** → **Pipelines**
* **既存の大規模 Agent Pool/Release（Classic）資産がある** → **Pipelines 継続**
* **ガバナンスはGitHub側で完結（Environments/Required reviewers/Branch protection）** → **Actions**

# 機能対応の要点（大差なし）

* **Azure 認証**：どちらも \*\*OIDC（Workload Identity Federation）\*\*で長期シークレット不要
* **マルチステージ/YAML**：両者とも対応（Plan→Apply、Build→Deploy など）
* **自己ホスト実行**：Actions＝Self-hosted Runner、Pipelines＝Self-hosted Agent（プール管理）
* **キャッシュ/マトリクス/並列**：両者とも可
* **レジストリ**：どちらも ACR への push/pull、関数・Container Apps 更新が可能
* **シークレット**：Actions＝Environments/Secrets、Pipelines＝Variables/Library＋Key Vault 連携（両者ともKey Vault連携可）

# あなたの構成に即したおすすめ

> フロント → **API Management** → **Functions（ingest）** → **Queue Storage** → **Container Apps（非同期ワーカー）**

* **CI/CD 基本線**：**GitHub Actions**

  * CI：lint / test / コンテナビルド → **ACR** へ push
  * CD：`az functionapp config container set` / `az containerapp update` で新リビジョン反映（必要ならカナリア）
  * 認証：Actions→Azure を **OIDC**、実行時秘密は **Key Vault**＋Managed Identity
* **Terraform**：同じく Actions で `plan` をPRに出力→承認後 `apply`（Stateは Azure Blob）
* **ガバナンス**：GitHub **Environments** と **Required reviewers**、保護ブランチ、署名付きイメージ/アーティファクト（必要に応じて）

# いつ Azure Pipelines を選ぶべきか（具体例）

* **Azure Boards と紐づいた変更要求・承認・デプロイ証跡**を一元管理したい
* **Release（Classic）で段階的配布・ゲート・手動承認フロー**を既に使っている
* **社内ネットワークからのみ実行**（インターネット到達なし）で CI/CD を回したい

# 実務的な指針

* どちらか**一方に統一**し、ログ・権限・秘密管理を一本化（“二重運用”は避ける）
* 切替コストがなければ **まず Actions**、強い理由があれば **Pipelines** を選択

**まとめ**：現状（GitHub 中心・Azure へデプロイ）の前提なら **Azure Pipelinesは不要**。**GitHub Actions**で CI/CD を組むのが合理的です。組織の変更管理を Azure DevOps に寄せる強い要件がある場合のみ **Pipelines** を検討してください。






以下、要点だけ端的にまとめます。

# OIDC（OpenID Connect）とは

* **OAuth 2.0の上に“認証（ログイン）”の層を追加した標準規格**です。
* ログイン結果を \*\*ID トークン（JWT）\*\*で渡し、ユーザーの“誰か（identity）”を検証可能にします。
* 代表的な発行者（**OpenID Provider/OP**）：**Microsoft Entra ID**、Google、Auth0 など。
* アプリ（**Relying Party/RP**）は、公開鍵（JWKS）で署名検証し、`iss / aud / exp / nonce` などの**クレーム**を確認します。

## トークンの役割

* **ID トークン**：本人確認用（誰がサインインしたか）。API呼び出しには原則使わない。
* **アクセス トークン**：APIを呼ぶ権限を示す（OAuth2 由来）。
* **リフレッシュ トークン**：長期セッション更新用（安全な場所でのみ保持）。

## 推奨フロー

* **Authorization Code + PKCE**（Web/SPAs/モバイルの標準）。
* 旧来の **Implicit Flow** は推奨されません。

---

# 他方式との違い（ざっくり）

* **OAuth 2.0**：権限委譲（“何ができるか”）。
* **OIDC**：本人確認（“誰か”）。＝OAuth2を拡張して**ログイン標準**にしたもの。
* **SAML**：XMLベースの古参規格。OIDCは**JSON/JWT**で軽量、Web/モバイルに親和。

---

# あなたの構成での OIDC の使われ方（2通り）

1. **フロントエンド → APIM → Functions（アプリのユーザーログイン）**

```
Browser
  │ (Auth Code + PKCE)
  ▼
Entra ID ──> ID Token / Access Token (JWT)
  │
  ▼
API Management（JWT検証、claims→ヘッダー転写）
  │
  ▼
Functions / Backend（必要に応じ OBO で下流API用トークン取得）
```

* **APIM** で `validate-...jwt` によりトークンを検証し、`oid/tid/roles` 等をバックエンドへ渡すのが定石。

2. **GitHub Actions → Azure（秘密レス CI/CD の“ワークロードID連携”）**

```
GitHub Actions (OIDC 発行) ──> Entra ID (federated credentials で検証)
                                      │
                                      └─> Azure 用アクセス トークン発行
```

* 人ではなく**ワークフロー自体**を“主体”として信頼。**長期シークレット不要**で `az`/Terraform 等が実行できます。

---

# 実務チェックリスト（最重要）

* [ ] **IDトークンは“本人確認用”**。API呼び出しは**アクセス トークン**を使用。
* [ ] `iss / aud / exp / iat / nbf / nonce` を**必ず検証**。特に `aud`（想定クライアント）と `iss`（発行者）。
* [ ] SPA/モバイルは **Code + PKCE**。Implicit は避ける。
* [ ] JWKS の**キー回転**を前提に実装（kid を確認）。
* [ ] APIM で受けた独自ヘッダーは**一度削除してから**、検証済みクレームで**上書き**（偽装防止）。
* [ ] CI/CD は \*\*OIDC（フェデレーテッド資格情報）\*\*でクラウドにログイン、**長期シークレットを排除**。

---

## 一言で

> **OIDC = “OAuth2に標準のログイン機能を足して、IDトークンで『誰か』を安全に伝える仕組み」。**
> ユーザーログイン（APIM/Functions）にも、秘密レスのCI/CD（GitHub→Azure）にも使えます。
