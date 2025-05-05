# Go 系統

## Go 的 runtime

Go 的 **runtime** 是 Go 語言執行時期系統的核心元件，負責管理程式在執行時的各種底層操作。它是一個內建的函式庫（runtime 套件），與 Go 程式一起編譯並嵌入到最終的可執行檔中，無需依賴外部執行時間環境。 Go 的 runtime 主要負責以下功能：

1. **協程（Goroutine）調度**：
    - Go 的 runtime 包含一個高效的調度器（scheduler），用於管理 goroutine 的創建、調度和執行。
    - 它採用 **M:N 調度模型**，將多個 goroutine（M）映射到少量作業系統線程（N）上運行，以實現輕量級並發。
    - 調度器會處理 goroutine 的切換、暫停、恢復，以及在多核心 CPU 上的負載平衡。
2. **記憶體管理**：
    - runtime 包含垃圾回收器（Garbage Collector, GC），用於自動管理記憶體分配和回收。
    - 它透過分代、並發垃圾回收機制減少記憶體碎片和程式暫停時間。
3. **系統呼叫和執行緒管理**：
    - runtime 負責與作業系統交互，處理底層系統呼叫（如檔案 I/O、網路操作）。
    - 它管理線程池，動態調整線程數量以優化效能，避免線程過度創建。
4. **並發模型支援**：
    - runtime 提供了 channel、select 等並發原語的底層實現，支援 Go 的 CSP（Communicating Sequential Processes）並發模型。
5. **其他功能**：
    - 提供執行時間除錯工具（如 runtime.Gosched()、runtime.Stack()）。
    - 處理訊號、異常、panic/recover 等運行時錯誤。
    - 管理程式的初始化、終止以及 CPU 和記憶體使用統計。

### 具體到協程調度

協程（goroutine）的調度由 Go runtime 的調度器（scheduler）全權負責。調度器運行在用戶態，避免了頻繁的作業系統內核態切換，從而提高了效能。具體調度過程包括：

- **工作竊取（Work-Stealing）**：當某個執行緒上的 goroutine 隊列為空時，調度器會從其他執行緒的佇列中「偷」 goroutine 來執行。
- **搶佔式調度**：在 Go 1.14 及以上版本，調度器支援搶佔式調度，避免某個 goroutine 長時間佔用線程。
- **事件驅動**：調度器會在特定事件（如 I/O 阻塞、系統呼叫、定時器到期）觸發 goroutine 切換。

### 總結

Go 的 runtime 是一個輕量、有效率的執行時間系統，負責協程調度、記憶體管理、執行緒管理等底層任務。它使得 Go 程式能夠以高並發、低開銷的方式運行，是 Go 語言「簡單而強大」的核心支撐。你可以將其理解為 Go 程式的“幕後管理者”，確保程式高效且穩定運作。

### Goroutine 與 Thread 的關係

Goroutine 是由 Go 運行時管理的輕量級執行緒

一個 Go 程式(Process)內可以有數千甚至數百萬個 Goroutine ，因為記憶體開銷小 （初始約 2KB）

相比作業系統 Thread（通常需要 1MB 以上堆疊）更高效，這允許 Go 在單一 Process 內創建大量 Goroutine

這些 Goroutine 會被映射到 Thread 上執行，Go 會使用內建的排程器實現併發或並行

### Go 的併發與並行

併發與並行取決於系統的核心數和 `GOMAXPROCS` 和執行的任務種類

CPU 只有單核心時， Go 會快速切換 Goroutine  實現併發

在併發時，當 Goroutine 執行 I/O 操作、系統主動呼叫、讓出控制權等，Go 排程器會調度其他 Goroutine 到當前 Thread 上

Go 用 `GOMAXPROCS` 來控制處理的數量，例如設置為 1 時，縱使是多核心仍然可以實現併發

目前 `GOMAXPROCS`  預設為系統的核心數

在多核心且 `GOMAXPROCS`  為預設時，Go 運行時會將 Goroutine 分發到多個作業系統 Thread，作業系統分配 Thread 在不同的 CPU 核心上運行，從而實現真正的並行

多核心處理的情況下，資源仍然是共用的，因為 goroutine 運行在同一個 Process 內，同一 Process 內的所有 Thread （以及映射到 Thread 的 Goroutine）共享相同的記憶體空間跟資源

遇到 I/O密集型任務時，即使在多核心系統上，如果 Goroutine 執行大量 I/O 操作，Go 運行時會傾向併發執行，因為會頻繁的阻塞和切換

如果遇到運算密集型任務，在多核心系統上通常會實現並行執行，充分利用多個核心

### Call stack

在 Go 中，每個 goroutine 都有自己的 call stack，獨立執行不會互相干擾函數的呼叫和局部變數

當一個 goroutine 發生 panic 時，只會打印出該 goroutine 的 call stack