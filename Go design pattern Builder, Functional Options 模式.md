# Go design pattern Builder, Functional Options 模式

### 簡易 builder

提供一個構造函數，去回傳一個實例

有時參數並不是必須的，需額外處理 `nil` 的邏輯

```go
type Server struct {
    Addr string
    Port int
    Conf *Config
}

type Config struct {
    Protocol string
    Timeout  time.Duration
    Maxconns int
    TLS      *tls.Config
}

func NewServer(addr string, port int, conf *Config) (*Server, error) {
    //...
}

func main() {

	srv1, _ := NewServer("localhost", 9000, nil) 
	conf := Config{Protocol:"tcp", Timeout: 60*time.Duration}
	srv2, _ := NewServer("locahost", 9000, &conf)

}

```

### 優化 builder

使用鏈式函數調用的方式來構造一個對象，必要參數定義在 `Server` ，非必要參數呼叫`ServerBuilder` 的方法做設定

雖然可直接將這些構造用在 `Server`，不需要在另外包一個 `ServerBuilder`

但如果以「單一職責原則」的角度來看

`Server` 結構表示一個服務器的配置，卻還承擔了構建階段的錯誤狀態，顯得不合理，一但 `Server` 構建成功，`err` 就變得冗餘，因為構建階段已結束

```go
//使用一个builder类来做包装
type ServerBuilder struct {
  Server
  err error
}
func (sb *ServerBuilder) Create(addr string, port int) *ServerBuilder {
  sb.Server.Addr = addr
  sb.Server.Port = port
  //其它代码设置其它成员的默认值
  return sb
}
func (sb *ServerBuilder) WithProtocol(protocol string) *ServerBuilder {
  sb.Server.Protocol = protocol 
  return sb
}
func (sb *ServerBuilder) WithMaxConn( maxconn int) *ServerBuilder {
  sb.Server.MaxConns = maxconn
  return sb
}
func (sb *ServerBuilder) WithTimeOut( timeout time.Duration) *ServerBuilder {
  sb.Server.Timeout = timeout
  return sb
}
func (sb *ServerBuilder) WithTLS( tls *tls.Config) *ServerBuilder {
  sb.Server.TLS = tls
  return sb
}
func (sb *ServerBuilder) Build() (Server) {
  return  sb.Server
}

func main() {
	sb := ServerBuilder{}
	server, err := sb.Create("127.0.0.1", 8080).
	  WithProtocol("udp").
	  WithMaxConn(1024).
	  WithTimeOut(30*time.Second).
	  Build()
}
```

### Functional Options

一個 Server 結構

```go
type Server struct {
    Addr     string
    Port     int
    Protocol string
    Timeout  time.Duration
    MaxConns int
    TLS      *tls.Config
}
```

```go
type Option func(*Server)

func Protocol(p string) Option {
    return func(s *Server) {
        s.Protocol = p
    }
}
func Timeout(timeout time.Duration) Option {
    return func(s *Server) {
        s.Timeout = timeout
    }
}
func MaxConns(maxconns int) Option {
    return func(s *Server) {
        s.MaxConns = maxconns
    }
}
func TLS(tls *tls.Config) Option {
    return func(s *Server) {
        s.TLS = tls
    }
}

// 建構
func NewServer(addr string, port int, options ...func(*Server)) (*Server, error) {
  srv := Server{
    Addr:     addr,
    Port:     port,
  }
  
  // options 都是執行函式後返回的，返回的函數記住了參數值
  for _, option := range options {
    option(&srv)
  }
  //...
  return &srv, nil
}

func main() {
	s1, _ := NewServer("localhost", 1024)
	s2, _ := NewServer("localhost", 2048, Protocol("udp"))
	s3, _ := NewServer("0.0.0.0", 8080, Timeout(300*time.Second), MaxConns(1000))
}
```

解說：

當你執行 `MaxConns(1000)` ，它返回一個待執行的函數，因著閉包特性，返回的函數記住了 `maxconns = 1000` 

將以上作為一個 型別為 function 的 參數的值 `options ...func(*Server)` 

在建構的時候，將 `options` 裡的函數（執行函數(ex. MaxConns) 的回傳的函數 ）全部執行

就能將呼叫建構函數`NewServer` 時，傳進各個執行函數的參數值，設置到 `Server` 的實例 s1, s2, s3

筆記參考資源：

[https://coolshell.cn/articles/21146.html](https://coolshell.cn/articles/21146.html)