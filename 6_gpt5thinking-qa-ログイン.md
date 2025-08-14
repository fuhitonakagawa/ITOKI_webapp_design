以下の組み合わせが、**「組織（Work/School）の MS アカウント SSO」**と**「フロントエンドへの WAF＋IP ホワイトリスト制限」**を最短で満たします。Azure 版と AWS 版を**一対一で対応**させて記します。

---

# Azure での推奨構成

## A. エッジで守る（推奨）

* **SSO（ログイン画面）**：Microsoft **Entra ID** でアプリ登録（OIDC / Authorization Code + PKCE、`authority=https://login.microsoftonline.com/organizations` で企業アカ限定）。フロントは MSAL を利用。([Microsoft Learn][1])
* **WAF＋IP 制限**：**Azure Front Door（Standard/Premium）＋ WAF**。**カスタム IP 制限ルール**で許可 IP のみ通過（CIDR 指定）。([Microsoft Learn][2])
* **フロントエンド実体**：**Azure App Service**（もしくは Static Web Apps / Storage 静的サイト）を **Front Door のオリジン**に。Premium なら **Private Link** で**オリジン非公開化**も可能。([Microsoft Learn][3])
* **バックエンド保護（任意）**：App Service の \*\*内蔵認証（Easy Auth）\*\*で OIDC 検証をオフロード可。([Microsoft Learn][4])

> メモ：実際のサインイン UI は Microsoft の認証エンドポイント側に遷移します。**ログイン自体の発生可否をネットワークで制御**したい場合は、\*\*Entra の条件付きアクセス（Named locations）\*\*で企業の送信元 IP に限定するのが堅実です（WAF と併用推奨）。([Microsoft Learn][5])

## B. リージョンで守る（代替）

* **WAF 層**：\*\*Application Gateway（WAF v2）\*\*で公開し、**WAF のカスタムルール**や **NSG** と併用して IP 許可。([Microsoft Learn][6])
* **アプリ**：**App Service** あるいは **AKS/Container Apps** を背後に。App Service 側でも **Access Restrictions** で IP 許可リストを併用可能。([Microsoft Learn][7])

## 主要設定ポイント（Azure）

* **アプリ登録**：サインイン対象を *Single-tenant*（自社のみ）または *Multitenant*（複数テナント）。**企業アカのみ**なら `organizations` を利用。([Microsoft Learn][1])
* **MSAL / OIDC**：Auth Code + PKCE を利用。ID/アクセス/更新トークンの取得と JWKS 検証はライブラリに準拠。([Microsoft Learn][8])
* **Front Door WAF**：**IP 制限のカスタムルール**を作成（許可リスト方式）。必要に応じて **Managed Rules** も有効化。([Microsoft Learn][2])
* **Private Link**（任意）：Front Door Premium から App Service/Storage を**プライベート接続**し、**インターネット非公開**運用。([Microsoft Learn][3])

---

# AWS での同等構成

## A. エッジで守る（推奨）

* **SSO（ログイン画面）**：**Amazon Cognito（User Pool）** をフロントに置き、**Azure Entra ID を OIDC IdP として連携**（ホスト型 UI で企業アカ サインイン）。([AWS ドキュメント][9], [Amazon Web Services, Inc.][10])
* **WAF＋IP 制限**：**Amazon CloudFront ＋ AWS WAF**。**IP set** を使い **許可 IP のみ許容**。Web ACL を CloudFront に関連付け。([AWS ドキュメント][11], [Repost][12])
* **フロントエンド実体**：

  * 静的サイト：**S3 + CloudFront（OAC）**
  * SSR/コンテナ：**ALB（ECS/EKS/EC2）** を CloudFront のオリジンに
  * API：**API Gateway**（必要ならステージに WAF 連携）([AWS ドキュメント][13])

## B. リージョンで守る（代替）

* **WAF 層**：**ALB + AWS WAF** をリージョンで関連付け、IP 許可リストで制御。([AWS CLI Command Reference][14], [AWS ドキュメント][15])
* **アプリ**：Amplify Hosting / App Runner / ECS など。App Runner も WAF 連携可。([AWS ドキュメント][16])

---

# サービス対応表（要点）

| 要件                   | Azure                                           | AWS                                                                                                   |
| -------------------- | ----------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| 企業 MS アカ SSO（OIDC）   | **Entra ID**（アプリ登録、`organizations` で企業限定）; MSAL | **Cognito User Pool** + **Entra ID(OIDC) 連携**（または直接 OIDC 実装）. ([Microsoft Learn][1], [AWS ドキュメント][9]) |
| エッジ WAF & IP ホワイトリスト | **Front Door + WAF**（カスタム IP ルール）               | **CloudFront + AWS WAF**（IP set/ACL）. ([Microsoft Learn][2], [AWS ドキュメント][11])                        |
| オリジン非公開化             | **Front Door Premium Private Link**             | **CloudFront + OAC（S3）** /（ALB は WAF/ヘッダ等で保護）. ([Microsoft Learn][3])                                 |
| アプリ側 IP 制限（補助）       | **App Service Access Restrictions**             | （区域 WAF/SG で補完）. ([Microsoft Learn][17])                                                              |
| 認証のオフロード             | **App Service Easy Auth**                       | Cognito Hosted UI / ALB OIDC（要設計）. ([Microsoft Learn][4])                                             |

---

# 設計の勘所（実務 Tips）

1. **ログイン可否を IP で縛るなら**：WAF 側のホワイトリストに加え、\*\*Entra 条件付きアクセス（Named locations）\*\*で「企業 IP 以外はサインイン不可」を併用すると二重で堅牢になります。([Microsoft Learn][5])
2. **マルチテナント受け入れ時**：Entra では *Accounts in any org directory* を選び、`organizations` エンドポイントで誘導。管理者同意フローの設計を忘れずに。([Microsoft Learn][18])
3. **SPA/SSR は Auth Code + PKCE が原則**（Implicit は不可）。([Microsoft Learn][8])
4. **Azure 側オプション**：エッジで遮断（Front Door WAF）→ オリジンは Private Link / Access Restrictions で**非公開化＋二重絞り**がベストプラクティス。([Microsoft Learn][3])
5. **AWS 側オプション**：CloudFront に WAF を関連付け、**IP set** で許可リスト化。必要に応じ **ALB / API Gateway** にも Web ACL を関連付けます。([AWS ドキュメント][19])

---

必要であれば、**Next.js（Azure Front Door + App Service + Entra）**または**CloudFront + Cognito（Azure AD 連携）**の**最小実装テンプレート**と、ポータルの設定手順（画面遷移・値）を**一気通貫**でお作りします。どちらのスタックで進めますか。

[1]: https://learn.microsoft.com/en-us/entra/identity-platform/v2-protocols-oidc?utm_source=chatgpt.com "OpenID Connect (OIDC) on the Microsoft identity platform"
[2]: https://learn.microsoft.com/en-us/azure/web-application-firewall/afds/waf-front-door-configure-ip-restriction?utm_source=chatgpt.com "Configure an IP restriction WAF rule for Azure Front Door"
[3]: https://learn.microsoft.com/en-us/azure/frontdoor/private-link?utm_source=chatgpt.com "Secure your Origin with Private Link in Azure Front Door ..."
[4]: https://learn.microsoft.com/en-us/azure/app-service/overview-authentication-authorization?utm_source=chatgpt.com "Authentication and Authorization - Azure App Service | Microsoft Learn"
[5]: https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-assignment-network?utm_source=chatgpt.com "Network in Conditional Access policy - Microsoft Entra ID"
[6]: https://learn.microsoft.com/en-us/answers/questions/1179573/how-to-define-ip-whitelist-in-azure-application-ga?utm_source=chatgpt.com "How to define IP whitelist in Azure Application Gateway ..."
[7]: https://learn.microsoft.com/en-us/azure/app-service/overview-access-restrictions?utm_source=chatgpt.com "Azure App Service access restrictions"
[8]: https://learn.microsoft.com/en-us/entra/identity-platform/msal-authentication-flows?utm_source=chatgpt.com "Authentication flow support in MSAL"
[9]: https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-oidc-idp.html?utm_source=chatgpt.com "Using OIDC identity providers with a user pool"
[10]: https://aws.amazon.com/blogs/security/how-to-set-up-amazon-cognito-for-federated-authentication-using-azure-ad/?utm_source=chatgpt.com "How to set up Amazon Cognito for federated authentication ..."
[11]: https://docs.aws.amazon.com/waf/latest/developerguide/waf-ip-set-managing.html?utm_source=chatgpt.com "Creating and managing an IP set in AWS WAF"
[12]: https://repost.aws/knowledge-center/waf-allow-my-ip-block-other-ip?utm_source=chatgpt.com "Use AWS WAF to restrict IPv4 and IPv6 access"
[13]: https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-control-access-aws-waf.html?utm_source=chatgpt.com "Use AWS WAF to protect your REST APIs in API Gateway"
[14]: https://awscli.amazonaws.com/v2/documentation/api/2.0.33/reference/wafv2/associate-web-acl.html?utm_source=chatgpt.com "associate-web-acl — AWS CLI 2.0.33 Command Reference"
[15]: https://docs.aws.amazon.com/waf/latest/developerguide/web-acl-associating-aws-resource.html?utm_source=chatgpt.com "Associating or disassociating protection with an AWS ..."
[16]: https://docs.aws.amazon.com/apprunner/latest/dg/waf.html?utm_source=chatgpt.com "Associating an AWS WAF web ACL with your service"
[17]: https://learn.microsoft.com/en-us/azure/app-service/app-service-ip-restrictions?utm_source=chatgpt.com "Set up Azure App Service access restrictions"
[18]: https://learn.microsoft.com/en-us/security/zero-trust/develop/identity-supported-account-types?utm_source=chatgpt.com "Identity and account types for single- and multitenant apps"
[19]: https://docs.aws.amazon.com/waf/latest/developerguide/cloudfront-features.html?utm_source=chatgpt.com "Using AWS WAF with Amazon CloudFront"
