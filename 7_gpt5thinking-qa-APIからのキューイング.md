結論：**できます。Function を必須で挟む必要はありません。**
フロントエンドで Entra ID による認証 → **API Management (APIM)** でトークン検証 → **APIM の Managed Identity** を使って **Azure Queue Storage の REST API（Put Message）** を直接呼び出す、という構成で **キュー投入をダイレクト**に実現できます。([Microsoft Learn][1])

---

## 最小構成とフロー

1. **APIM でフロントの JWT を検証**（`validate-jwt` ポリシー。`iss`/`aud` などを固定）。([Microsoft Learn][1])
2. **APIM に Managed Identity を付与**し、対象ストレージ（または対象キュー）に **RBAC: *Storage Queue Data Message Sender*** を割り当て。([Microsoft Learn][2])
3. APIM のポリシーで

   * `authentication-managed-identity`（リソース: `https://storage.azure.com/`）で **Bearer トークンを自動取得**し、
   * `send-request` で **Queue Storage の `.../messages` に POST**（Put Message）します。([Microsoft Learn][3])
   * Queue のメッセージ本文は XML。**JSON をそのまま入れるなら XML エスケープ**、手堅くやるなら **Base64 化**して `<MessageText>` に入れます。([Microsoft Learn][4])

### ポリシー例（概略）

```xml
<inbound>
  <base />
  <!-- Entra ID トークン検証 -->
  <validate-jwt header-name="Authorization" failed-validation-httpcode="401">
    <openid-config url="https://login.microsoftonline.com/{tenant}/v2.0/.well-known/openid-configuration" />
    <audiences>
      <audience>api://your-api-app-id</audience>
    </audiences>
  </validate-jwt>

  <!-- リクエスト本文を Base64 に（必要に応じて） -->
  <set-variable name="body" value="@((string)context.Request.Body.As<string>(preserveContent:true))" />
  <set-variable name="b64" value="@{
      var s = (string)context.Variables["body"];
      return Convert.ToBase64String(System.Text.Encoding.UTF8.GetBytes(s));
  }" />

  <!-- Queue Storage へ送信 -->
  <send-request mode="new" response-variable-name="qres" timeout="10">
    <set-url>@($"https://{storageAccount}.queue.core.windows.net/{queueName}/messages")</set-url>
    <set-method>POST</set-method>
    <set-header name="x-ms-version" exists-action="override">
      <value>2023-11-03</value>
    </set-header>
    <authentication-managed-identity resource="https://storage.azure.com/" />
    <set-body>@($"<QueueMessage><MessageText>{context.Variables["b64"]}</MessageText></QueueMessage>")</set-body>
  </send-request>

  <return-response>
    <set-status code="201" reason="Created" />
  </return-response>
</inbound>
```

（`authentication-managed-identity` と `send-request` の組み合わせは公式にサポート。APIM のポリシー式は C# で `Convert.ToBase64String` 等が使えます。）([Microsoft Learn][3])

---

## Function を挟むのはいつ？

**要件が簡素（“受け取ったペイロードをそのままキューへ”）なら APIM 直送で十分**です。
一方、下記のような場合は **Azure Functions（または Logic Apps）を挟む**方が保守性・拡張性が高くなります。

* **スキーマ検証／正規化／マスキング**などのビジネスロジックを実装したい
* **リトライ／冪等性**、相関 ID 付与、監査ログ付与などの**複合処理**が必要
* **別サービスへの多重書き込み**や条件分岐をしたい
* 将来的に **Service Bus（FIFO/セッション/重複検出/DLQ など）**へ切替え・併用する可能性がある
  （APIM は“ゲートウェイ”としての**軽量変換・認証/保護**に留め、**複雑なロジックはバックエンドへ**が推奨です。）([Microsoft Learn][5], [serverlessnotes.com][6])

> 参考：**Storage Queue と Service Bus Queue の比較**（機能要件によって使い分け）— FIFO/セッション/重複検出/DLQ 等が要るなら Service Bus を検討。([Microsoft Learn][7])

---

## 補足チェックリスト

* ストレージ側ネットワーク制限を有効にしている場合、**APIM からの到達性**（VNet/Private Endpoint 経路など）を確認
* **RBAC は最小権限**で（送信だけなら *Storage Queue Data Message Sender*）([Microsoft Learn][8])
* **`x-ms-version` を明示**し、**本文は XML 形式**（必要なら Base64）で送信。([Microsoft Learn][4])

---

必要でしたら、**実環境名を埋めた APIM ポリシー（動作確認済み）**と、Portal での**役割割り当て手順**をセットでお渡しします。

[1]: https://learn.microsoft.com/en-us/azure/api-management/validate-jwt-policy?utm_source=chatgpt.com "Azure API Management policy reference - validate-jwt"
[2]: https://learn.microsoft.com/en-us/azure/api-management/api-management-howto-use-managed-service-identity?utm_source=chatgpt.com "Use Managed Identities in Azure API Management | Microsoft Learn"
[3]: https://learn.microsoft.com/en-us/azure/api-management/authentication-managed-identity-policy?utm_source=chatgpt.com "authentication-managed-identity - Learn Microsoft"
[4]: https://learn.microsoft.com/en-us/rest/api/storageservices/put-message "Put Message (REST API) - Azure Storage | Microsoft Learn"
[5]: https://learn.microsoft.com/en-us/azure/well-architected/service-guides/azure-api-management?utm_source=chatgpt.com "Architecture Best Practices for Azure API Management"
[6]: https://www.serverlessnotes.com/docs/five-best-practices-when-working-with-azure-api-management?utm_source=chatgpt.com "Five best practices when working with Azure API ..."
[7]: https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-azure-and-service-bus-queues-compared-contrasted?utm_source=chatgpt.com "Compare Azure Storage queues and Service Bus queues"
[8]: https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles/storage?utm_source=chatgpt.com "Azure built-in roles for Storage"
