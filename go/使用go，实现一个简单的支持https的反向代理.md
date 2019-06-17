Proxy继承自httputil.ReverseProxy，并且增加了相应的处理https的ServeHTTPS方法

```go
package main

import(
  "fmt"
  "io"
  "log"
  "net"
  "net/http"
  "net/http/httputil"
  "sync"
)

// Proxy reverse proxy inherit from httputil.ReverseProxy and add https support
type Proxy struct {
  httputil.ReverseProxy
}

func (p *Proxy) ServeHTTP(w http.ResponseWriter, r *http.Request) {
  fmt.Println(r.URL)
  
  if r.Method == http.MethodConnect {
    p.ServeHTTPS(w, r)
  } else {
    if !r.URL.IsAbs() {
      handleError(w, "proxy error: not respond to non-proxy requests", http.StatusBadRequest)
    } else {
      p.ReverseProxy.ServeHTTP(w, r)
    }
  }
}

// ServeHTTPS handle https proxy
func (p *Proxy) ServeHTTPS(w http.ResponseWriter, r *http.Request) {
  hijack, ok := w.(http.Hijacker)
  if !ok {
    handleError(w, "proxy error: not support hijack", http.StatusBadGateway)
    return
  }
  oConn, oBufrw, err := hijack.Hijack()
  if err != nil {
    handleError(w, fmt.Sprintf("proxy error: %v", err.Error()), http.StatusBadGateway)
    return
  }
  tConn, err := net.Dial("tcp", r.Host)
  if err != nil {
    handleError(w, fmt.Sprintf("proxy error: %v", err.Error()), http.StatusBadGateway)
    return
  }
  oBufrw.WriteString("HTTP/1.0 200 OK\r\n\r\n")
  oBufrw.Flush()
  oTCPConn, ok1 := oConn.(*net.TCPConn)
  tTCPConn, ok2 := tConn.(*net.TCPConn)
  if ok1 && ok2 {
    go pipeTCPConn(oTCPConn, tTCPConn)
    go pipeTCPConn(tTCPConn, oTCPConn)
  } else {
    wg := &sync.WaitGroup{}
    wg.Add(2)
    go pipeConn(oConn, tConn, wg)
    go pipeConn(tConn, oConn, wg)
    wg.Wait()
    oConn.Close()
    tConn.Close()
  }
}

func handleError(w http.ResponseWriter, err string, code int) {
  fmt.Println(err)
  http.Error(w, err, code)
}

func pipeTCPConn(src , dst *net.TCPConn) {
  if _, err := io.Copy(dst, src); err != nil {
    fmt.Println(err)
  }
  src.CloseRead()
  dst.CloseWrite()
}

func pipeConn(src, dst net.Conn, wg *sync.WaitGroup) {
  if _, err := io.Copy(dst, src); err != nil {
    fmt.Println(err)
  }
  wg.Done()
}

func proxy() *Proxy {
  return &Proxy{
    ReverseProxy: httputil.ReverseProxy{
      Director: func (req *http.Request) {

      },
    },
  }
}

func main() {
  log.Fatal(http.ListenAndServe(":8080", proxy()))
}
```