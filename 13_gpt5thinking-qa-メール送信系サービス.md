承知しました。まずは **AWS の SNS/SMS/SES に相当する Azure のサービス対応表**、続いて「処理完了メール」を送る実装パターンをご提示します。

# 対応表（AWS ⇔ Azure）

| 目的                       | AWS                    | Azure（推奨）                                                      | 補足                                                                                         |
| ------------------------ | ---------------------- | -------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| **メール送信（トランザクション/通知）**   | **SES**                | **Azure Communication Services (ACS) – Email**                 | 独自/管理ドメインを関連付けてスケーラブルに送信。SDK/REST あり。Logic Apps からコネクタも利用可能。 ([Microsoft Learn][1])        |
| **SMS 送信**               | **SNS（SMS）/ Pinpoint** | **ACS – SMS**                                                  | A2P/SMS 用 SDK。ユースケースはリマインダや2FAなど。 ([Microsoft Learn][2])                                   |
| **モバイル Push**            | **SNS Mobile Push**    | **Azure Notification Hubs**                                    | APNs/FCM/WNS へ大規模配信するプッシュ基盤。 ([Microsoft Learn][3])                                        |
| **Pub/Sub（イベント・ファンアウト）** | **SNS Topics**         | **Azure Event Grid（イベント駆動）** / **Service Bus Topics（メッセージ指向）** | 目的で使い分け：外形イベントのファンアウト＝Event Grid、厳密なメッセージング/順序/TTL/再配信＝Service Bus。 ([Microsoft Learn][4]) |

> 注：Microsoft 365 テナントをお持ちで「ユーザーとして送信」したい場合は **Microsoft Graph の sendMail** も選択肢ですが、汎用のトランザクションメールには **ACS Email** が第一候補です。 ([Microsoft Learn][5])

---

# 「処理完了メール」を送るお勧め構成

## パターンA（運用性重視・ノーコード連携）

**Container App → Event Grid（カスタムトピック）→ Logic Apps → ACS Email**

* ワーカーは「完了イベント」を **Event Grid** に発行
* **Logic Apps** がトリガされ、テンプレート/条件分岐/宛先リスト管理を行い、**ACS Email コネクタ**で送信
* メリット：テンプレート運用・宛先管理・再送/分岐が容易。IaC で組みやすく監査も明快。
* 参考：**Event Grid** / **Logic Apps の ACS Email コネクタ**。 ([Microsoft Learn][4])

## パターンB（レイテンシ最小・コード直書き）

**Container App（ワーカー）から直接 ACS Email SDK/REST を呼ぶ**

* 仕組みが単純で低遅延
* マルチテナントなら **テナント別 From ドメイン**・テンプレート切替をアプリ側で実装
* 参考：**ACS Email のクイックスタート/EmailClient API**。 ([Microsoft Learn][6])

> どちらのパターンでも、**Queue Storage からの処理完了**は既存フローの中でフック可能です。通知の将来拡張（SMS/Push/Teams 連携）まで見据えるなら **パターンA** が拡張しやすいです。

---

# 最小コード例（パターンB：Python／ACS Email）

```python
# pip install azure-communication-email
import os
from azure.communication.email import EmailClient, EmailContent, EmailAddress, EmailRecipients, EmailMessage

conn = os.environ["ACS_EMAIL_CONNECTION_STRING"]  # ACS の接続文字列
client = EmailClient.from_connection_string(conn)

def send_done_mail(to_addr: str, job_id: str, download_url: str):
    subject = f"[完了] ジョブ {job_id}"
    html = f"<p>ジョブ {job_id} が完了しました。</p><p><a href='{download_url}'>結果を確認</a></p>"

    message = EmailMessage(
        sender=os.environ["ACS_FROM_ADDRESS"],  # 例: "no-reply@yourdomain.com"
        content=EmailContent(subject=subject, html=html),
        recipients=EmailRecipients(to=[EmailAddress(email=to_addr)])
    )
    client.send(message, disable_tracking=False)  # 非同期キュー投函型の送信
```

* **接続前提**：ACS リソース作成・**Email ドメインの関連付け/検証**（Azure 管理ドメイン or 独自ドメイン）。 ([Microsoft Learn][1])

---

# 設計の要点（マルチテナント想定）

* **通知経路の抽象化**：`notify(type=email|sms|push, template, tenant_id, params)` の Facade を用意して、裏で **ACS Email/SMS / Notification Hubs** を切替。
* **ドメイン運用**：テナント固有の From アドレス/ドメインが必要なら、**ACS Email に複数ドメイン登録**し、テナント設定DBでルーティング。 ([Microsoft Learn][1])
* **イベント駆動**：完了時は **Event Grid** に `JobCompleted` を発行し、通知だけでなく分析/監査フローにもファンアウト可能。 ([Microsoft Learn][4])
* **可観測性**：送信結果や失敗は **Logic Apps/ACS の結果**を Log Analytics に集約（再送ルール/アラート設定）。

---

## まとめ

* **AWS SES ⇔ Azure は ACS Email**、**SNS（SMS/Push）⇔ Azure は ACS SMS / Notification Hubs**、**SNS Topics ⇔ Event Grid / Service Bus Topics**。 ([Microsoft Learn][1])
* 処理完了メールは、**Event Grid → Logic Apps → ACS Email（運用性）** か \*\*ワーカーから ACS Email 直呼び（低遅延）\*\*が定石です。 ([Microsoft Learn][4])

必要でしたら、現行の **Queue Storage → Container App** のフローに最小変更で組み込む設計図（リソース図と IaC スニペット）も即時ご用意します。

[1]: https://learn.microsoft.com/en-us/azure/communication-services/concepts/email/email-overview?utm_source=chatgpt.com "Overview of Azure Communication Services email"
[2]: https://learn.microsoft.com/en-us/azure/communication-services/quickstarts/sms/send?utm_source=chatgpt.com "Send an SMS message - Azure Communication Services"
[3]: https://learn.microsoft.com/en-us/azure/notification-hubs/notification-hubs-push-notification-overview?utm_source=chatgpt.com "What is Azure Notification Hubs?"
[4]: https://learn.microsoft.com/en-us/azure/event-grid/?utm_source=chatgpt.com "Azure Event Grid documentation"
[5]: https://learn.microsoft.com/en-us/azure/communication-services/?utm_source=chatgpt.com "Azure Communication Services documentation"
[6]: https://learn.microsoft.com/en-us/azure/communication-services/quickstarts/email/send-email?utm_source=chatgpt.com "Send an email using Azure Communication Services"
