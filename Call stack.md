# Call stack

用於追蹤程式執行時的函式呼叫鏈，主要用於：

- 追蹤函數的執行順序：透過彈出或持續堆疊來表示出函數執行的順序關係或是巢狀關係
- 管理函數的局部變數和參數：每個函數的 stack frame 都為其局部變數和參數提供獨立的空間
- 處理函數的返回：函數執行完畢後，call stack 的返回地址會告訴程式接下來要執行哪裡
- 錯誤處理和追蹤：當程式發生錯誤時，會打印出 call stack 的資訊，這個資訊顯示錯誤發生的路徑和上下文，可以了解是哪個函數導致錯誤，或是由哪個函式呼叫的鏈式關係，依此往前回溯

## 順序執行

對於沒有直接呼叫關係的函數，它們在 call stack 中會**先後出現，然後再被彈出**

每個函數在執行期間都會有自己的 stack frame 存在於 call stack 的頂部。當一個函數執行完畢後，它的 stack frame就會消失，call stack 會回到呼叫它的函數的狀態

```go
func main() {
	funcA()
	funcB()
}
```

1. 程式剛開始執行 `main` 函數：此時，call stack 的底部是 `main` 函數的 stack frame

```go
[ main ] <--- 棧頂
```

2. `main` 函數呼叫 `funcA`：當 `funcA` 被呼叫時，它的 stack frame 會被壓入 call stack 的頂部

```go
[ funcA ] <--- 棧頂
[ main      ]
```

3. `funcA` 執行完畢並返回 `main` 函數：

`funcA` 執行完畢，它的 stack frame 會從 call stack 中彈出。現在，`main` 函數的 stack frame 又回到了棧頂，程式會回到 `main` 函數中 `funcA` 被呼叫之後的位置繼續執行

```go
[ main ] <--- 棧頂
```

**4. `main` 函數呼叫 `funcB`：**`main` 函數呼叫 `funcB`，`funcB` 的 stack frame 被壓入 call stack 的頂部

```go
[ funcB ] <--- 棧頂
[ main      ]
```

## 巢狀執行

當一個函數呼叫另一個函數，被呼叫的函數的 stack frame 會壓入 call stack 頂部，形成堆疊

當函數返回後，其 stack frame 彈出，控制權回到上一層的函數

```go
func funcA() {
	funcB()
}

func main() {
	funcA()
}
```

1. 程式剛開始執行 `main` 函數：此時，call stack 的底部是 `main` 函數的 stack frame

```go
[ main ] <--- 棧頂
```

2. `main` 函數呼叫 `funcA`：當 `funcA` 被呼叫時，它的 stack frame 會被壓入 call stack 的頂部

```go
[ funcA ] <--- 棧頂
[ main      ]
```

**3. `funcA`函數呼叫 `funcB`：**`main` 函數呼叫 `funcB`，`funcB` 的 stack frame 被壓入 call stack 的頂部

```go
[ funcB ] <--- 棧頂
[ funcA ]
[ main      ]
```

4.**`funcB`函數執行完畢:** `funcB` 的 stack frame 彈出，現在 `funcA` 恢復執行

```go
[ funcA ] <--- 棧頂
[ main      ]
```

5.**`funcA`函數執行完畢:** `funcA` 的 stack frame 彈出，現在 `main` 恢復執行

```go
[ main      ]
```