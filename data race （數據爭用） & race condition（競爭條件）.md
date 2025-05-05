# data race （數據爭用） & race condition（競爭條件）

### data race（數據爭用）

定義：多個線程對同一個變量，同時地進行讀寫操作，不包含所有線程只進行讀操作。

後果：該變數最後的值不可預測，使得併發編程結果不可預測

防止:   使用鎖

### race condition （競爭條件）

定義：受多併發線程執行的順序和時機影響造成的問題

後果：運行數次可能會有不一樣的結果

防止：通常是因為對一個資源的操作沒有符合 ACID原則中的 原子性、一致性、事務隔離，使用臨界區來解決問題

## 演示

data race

兩個線程都想操作 src, dst，無法保證順序的情況下，不知道結果會如何

```go
my_account := 0
your_account := 100

func racy_transfer(src *int, dst *int, m int) {
  if (m <= *src) {  //無法預測哪個線程先執行
    *src -= m      //無法預測哪個線程先執行
    *dst += m      //無法預測哪個線程先執行
    return true;
  } else {
    return false;
  }
}

go racy_transfer(&your_account, &my_account, 50);
go racy_transfer(&your_account, &my_account, 80);
```

race condition

多線程的代碼交錯執行的時機，符合條件進入 if 塊時，其他線程已經改變作為判斷標準的值，導致邏輯、限制本身就發生了不可預期的情況

```go
my_account := 0
your_account := 100

func racy_transfer(src *int, dst *int, m int) {
  if (m <= *src) { 
	  // 這個時候 m <= *src 是否仍成立？有沒有其他線程可能已經改變 src 的值了
    *src -= m      
    *dst += m      
    return true;
  } else {
    return false;
  }
}

go racy_transfer(&your_account, &my_account, 50);
go racy_transfer(&your_account, &my_account, 80);
```

解決方案

```go
my_account := 0
your_account := 100

var mutex sync.Mutex
func racy_transfer(src *int, dst *int, m int) {
	mutex.Lock()
	// 臨界區確保一次只能由一個 goroutine 存取
  if (m <= *src) { 
    *src -= m      
    *dst += m      
    return true;
  } else {
    return false;
  }
  mutex.UnLock()
}

go racy_transfer(&your_account, &my_account, 50);
go racy_transfer(&your_account, &my_account, 80);
```

## 針對變數的ACID操作

`sync/atomic` 套件提供了**底層的ACID記憶體操作**。操作是不可分割的，在執行過程中不會被其他 goroutine 中斷

支援的資料類型 `int32`、`int64`、`uint32`、`uint64` 和 `uintptr` 等基本資料類型，也支援對指標的原子操作

適用場景有計數器、標誌、狀態變數等

```go
var (
	counter int64 // 使用 int64，因為 atomic 套件常用這個類型
	wg      sync.WaitGroup
)

func incrementCounter() {
	defer wg.Done()

	for i := 0; i < 1000; i++ {
		atomic.AddInt64(&counter, 1) // 原子地增加計數器
		time.Sleep(time.Microsecond)
	}
}

func main() {
	wg.Add(2)
	go incrementCounter()
	go incrementCounter()

	wg.Wait()
	finalCounter := atomic.LoadInt64(&counter) // 原子地讀取最終計數器值
	fmt.Println("最終計數器值:", finalCounter)
}
```

多個 goroutine 同時存取並修改同一個計數器變數時，可能會發生 race condition ，導致最終值不正確

計數器變數 `counter`，最後希望 `counter = 2000`，兩個 goroutine 都執行以下操作 ：

1. 讀取 `counter` 的值
2. 將讀取到的值加 1
3. 將結果寫回 `counter` 

可能會發生兩個 goroutine 讀取一樣的值，並增加值上去，變成覆蓋行為

goroutine 1 讀取 counter (假設是 10)
goroutine 2 讀取 counter (也是 10)
goroutine 1 將 10 加 1，得到 11
goroutine 2 將 10 加 1，也得到 11
goroutine 1 將 11 寫回 counter
goroutine 2 將 11 寫回 counter  // 兩個讀取跟增加應該要增加2，最後只加了 1

`sync/atomic` 使你可以ACID的操作數字類型

1. 讀取 `counter` 的值
2. 將讀取到的值加 1
3. 將結果寫回 `counter` 

讓這三個步驟是ACID的，不會被其他 goroutine 穿插執行在中間