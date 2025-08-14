はい。**Azure Blob Storage は既定でサーバー側暗号化（SSE）されます**。アプリ側の変更なしに保存時に自動暗号化・読取時に自動復号が行われ、**無効化はできません**。([Microsoft Learn][1])

## S3との対応関係（ざっくり）

* **SSE-S3（AWS）** ≒ **Microsoft-managed keys（Azure既定）**
  何もしなくても AES-256 で暗号化されます。([Microsoft Learn][1])
* **SSE-KMS（AWS）** ≒ **Customer-managed keys / CMK（Key Vault/Managed HSM）**
  自分の鍵で暗号化を管理できます（キーローテーションやアクセス制御を強化）。([Microsoft Learn][2])
* **SSE-C（AWS）** ≒ **Customer-provided keys / CPK（要求ごとに鍵を持ち込み）**
  リクエストに鍵を添えて暗号化する方式（用途限定）。([Microsoft Learn][3])

また、Blob には**Encryption Scope**（コンテナ／Blob単位で鍵を切り替える仕組み）もあり、同一アカウント内でデータ境界を明確化できます。([Microsoft Learn][4])

> 参考：S3は2023年1月以降、全オブジェクトを自動暗号化するのが既定です（SSE-S3）。Azureの既定と同趣旨です。([AWS ドキュメント][5])

## 実務で押さえるポイント

* 既定の **Microsoft-managed keys** でも十分に保護されますが、**規制・内部統制**がある場合は **CMK（Key Vault/HSM）** を選択。([Microsoft Learn][2])
* **Secure transfer required（HTTPS強制）**、**Private Endpoint**、**RBAC** を併用して経路・アクセスも防御。
* 鍵をCMKにする場合は **Key Vault のソフト削除＋パージ保護**を有効化してから紐づけます。([Microsoft Learn][6])

要するに、**“S3と同様に自動暗号化される”**が正しく、必要に応じて**自社管理鍵やスコープ**で一段深い管理が可能、という理解で大丈夫です。

[1]: https://learn.microsoft.com/en-us/azure/storage/common/storage-service-encryption?utm_source=chatgpt.com "Azure Storage encryption for data at rest"
[2]: https://learn.microsoft.com/en-us/azure/storage/common/customer-managed-keys-overview?utm_source=chatgpt.com "Customer-managed keys for Azure Storage encryption"
[3]: https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blob-customer-provided-key?utm_source=chatgpt.com "Specify a customer-provided key on a request to Blob ..."
[4]: https://learn.microsoft.com/en-us/azure/storage/blobs/encryption-scope-overview?utm_source=chatgpt.com "Encryption scopes for Blob storage - Azure"
[5]: https://docs.aws.amazon.com/AmazonS3/latest/userguide/default-encryption-faq.html?utm_source=chatgpt.com "Default encryption FAQ - Amazon Simple Storage Service"
[6]: https://learn.microsoft.com/en-us/azure/storage/common/customer-managed-keys-configure-existing-account?utm_source=chatgpt.com "Configure customer-managed keys in the same tenant for ..."
