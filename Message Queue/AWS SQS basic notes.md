# AWS SQS

## Producers

通常是 aplication端，負責 send message 至 message queue 中

## redundant infra

分佈式 server，存放 message queue 上的 message，SQS 將 message 複製多份放到不同的 server 上，並不會每一台 server 都會放置所有的 message，這樣的機制即使 server 出現問題無法服務，也能從其他 server 中取得其副本，提高了 Availability(可用性)

## Consumers

負責將 message queue  pull message 下來處理，可以有多台 consumer 處理不同的 message 

message 被 consumers 處理時，其在 queue 狀態會變成 invisible，避免被其他 consumer 拉取下來處理在同一時間，當 consumer 處理完後即回報 queue 刪除所有 SQS server 上的 message 副本，才將 queue 上的 message 移除

pull 可調整其行為，例如一次拉取三筆 message 下來處理 (Batch Size)，亦或是可調整當 pull request 至 queue 可等待數秒，如果一開始 queue 中沒有 message，但在等待數秒時，message 進到 queue 時，即可拉取下來處理，這些目的都是為了減少 network request 提高效能，減少成本

## 要注意的重點
### 無法保證順序 （no order）

當 pull 拉取時，由 queue 負責分派要去哪一台 sqs server 拿取 message ，但該機制不會讓每一台 SQS server 都存放每一個 message 副本，因此分派到哪一台 server 即處理那台當前有的 message 副本，SQS 不保證處理順序能依 message 到達 queue 的順序

### SQS At-Least-Once Model 即有可能重複處理

當 message 被處理完、即刪除所有 sqs server 上的副本，但刪除有可能失敗，導致 message 無法順利被 queue 移除，進而讓 Consumer 再重新處理一次，即 delivery mode is at least onse （交付方式是至少一次）

必須確保 logic do it idempotent design (冪等設計)

### Visibility Timeout

consumer 拉取 Message 處理有時間限制，當處理超時時，queue 上該 message 的 invisible 狀態解除，該 message 可被其他 consumer 拉取處理，原處理超時的 consumer 繼續處理，即有可能一個 message 被處理數次
但萬一所有 consumer 處理該 message 都會超時怎麼辦？不就變成該 message 一直被重複執行

### Dead Letter Queue (DLQ)

當 visibility timeout 發生時，將 message 移到這個 queue 上，原拉取處理的 consumer 繼續處理，即有可能成功有可能失敗
分析 DLQ 上 message 是否有成功以及失敗的原因，也可重新發送 message