結論：**「SQS Standard 相当で十分」なら *Azure Queue Storage*、FIFO／重複排除やDLQなどの**メッセージング機能が要るなら ***Azure Service Bus*** が適任です。([Microsoft Learn][1])

---

## 使い分け（SQS 基準での早見表）

| 要件/性質                 | Queue Storage                         | Service Bus                                                                  |
| --------------------- | ------------------------------------- | ---------------------------------------------------------------------------- |
| 近いもの                  | **SQS Standard**（高スループット／順序はベストエフォート） | **SQS FIFO 相当**に近い設計が可能（**Sessions**で順序、**Duplicate detection**で重複排除）＋さらに高機能 |
| 順序保証                  | なし（ベストエフォート）                          | **Sessions**でキー単位の順序処理可                                                      |
| 重複排除 / “exactly-once” | なし                                    | **Duplicate detection**により**指定時間窓での**“exactly-once”を実現可                      |
| DLQ                   | 明示的な DLQ なし（実装でポイズン処理）                | **内蔵 DLQ** あり                                                                |
| 予約配信（遅延送信）            | なし                                    | **Scheduled delivery** あり                                                    |
| Pub/Sub               | なし                                    | **Topics & Subscriptions** あり                                                |
| メッセージサイズ              | **最大 64 KB**                          | **Standard: 256 KB／Premium: 最大 100 MB**（2024 以降の上限）                          |
| 価格感                   | **一般に安価**（ストレージ＋リクエスト課金）              | 機能に応じて **Basic/Standard/Premium**（高機能ほど高価）                                   |

（順序・重複排除・DLQ・予約配信・Topics などの有無やサイズ上限は公式仕様に基づきます。）([Microsoft Learn][1], [Microsoft Azure][2])
（Queue Storage の 64 KB 制限と TTL 仕様は公式ドキュメントに明記。）([Microsoft Learn][3])
（Service Bus の **Duplicate detection** は「指定ウィンドウ内の exactly-once」を提供。）([Microsoft Learn][4])
（料金体系の傾向：Queue Storage は“ストレージ＋操作課金”、Service Bus はティアごとに機能と単価が上がる。）([Microsoft Azure][5])

---

## おすすめ判断基準

* **とにかく「SQS Standard っぽい簡易キュー」で十分**
  → **Queue Storage**。高スループット・低コスト・運用シンプル。メッセージは 64 KB 以内（超える場合は **Blob へ本体**＋キューは参照だけ＝“**Claim Check**”パターン）。([Microsoft Learn][1])

* **順序・重複排除・DLQ・予約配信・Pub/Sub など“メッセージブローカー機能”が欲しい**
  → **Service Bus**（まずは Standard、**大容量やネットワーク分離**が要るなら Premium）。([Microsoft Learn][6])

---

## あなたの直近ユースケースへの適用

> **「フロント → Entra 認証 → APIM → キューに貯める」**
> ＝ビジネスロジックほぼなし・非同期投入だけ

* **要件がこの範囲に収まるなら**：まず **Queue Storage** を推奨。

  * APIM からは **Managed Identity** に ***Storage Queue Data Message Sender*** を付与し、REST の `Put Message` を呼ぶだけで直送可能。([Microsoft Learn][7], [azadvertizer.net][8])
* **将来**、厳密順序・DLQ・予約配信・Pub/Sub が必要になったら **Service Bus** へ。APIM からは ***Azure Service Bus Data Sender*** で同様に直送可能です。([Microsoft Learn][9])

---

## メッセージサイズの注意

* **Queue Storage**：本文 **64 KB**（Base64 化で実効サイズ増）。超えるなら **Blob へ本体**＋**キューはポインタ**に。([Microsoft Learn][3])
* **Service Bus**：**Standard 256 KB／Premium 最大 100 MB**。巨大ペイロード頻発なら Premium か **Claim Check** を検討。([Microsoft Learn][10])

---

### まとめ

* **“一般的な SQS（Standard）っぽく”で十分 → *Queue Storage***
* **順序・重複排除・DLQ・予約配信・Pub/Sub が欲しい／将来拡張を見据える → *Service Bus***

必要でしたら、**APIM ポリシーのサンプル**（Queue Storage／Service Bus 両対応）と、**RBAC 割当て手順**を実環境名入りでお渡しします。

[1]: https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-azure-and-service-bus-queues-compared-contrasted?utm_source=chatgpt.com "Compare Azure Storage queues and Service Bus queues"
[2]: https://azure.microsoft.com/en-us/pricing/details/service-bus/?utm_source=chatgpt.com "Service Bus pricing"
[3]: https://learn.microsoft.com/en-us/azure/storage/queues/storage-queues-introduction?utm_source=chatgpt.com "Introduction to Azure Queue Storage"
[4]: https://learn.microsoft.com/en-us/azure/service-bus-messaging/enable-duplicate-detection?utm_source=chatgpt.com "Enable duplicate message detection - Azure Service Bus"
[5]: https://azure.microsoft.com/en-us/pricing/details/storage/queues/?utm_source=chatgpt.com "Queue Storage pricing"
[6]: https://learn.microsoft.com/en-us/azure/service-bus-messaging/advanced-features-overview?utm_source=chatgpt.com "Azure Service Bus messaging - advanced features"
[7]: https://learn.microsoft.com/en-us/azure/storage/queues/assign-azure-role-data-access?utm_source=chatgpt.com "Assign an Azure role for access to queue data"
[8]: https://www.azadvertizer.net/azrolesadvertizer/c6a89b2d-59bc-44d0-9896-0f6e12d7b80a.html?utm_source=chatgpt.com "Storage Queue Data Message Sender - c6a89b2d-59bc- ..."
[9]: https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-authentication-and-authorization?utm_source=chatgpt.com "Azure Service Bus authentication and authorization"
[10]: https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-quotas?utm_source=chatgpt.com "Azure Service Bus - Messaging quotas"
