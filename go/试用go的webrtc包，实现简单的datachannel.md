```go
package main

import (
  "log"
  "os"

  "github.com/pion/webrtc"
)

// Peer webrtc端节点
type Peer struct {
  conn *webrtc.PeerConnection
  data *webrtc.DataChannel
  buff chan string
}

// NewPeer 创建Peer节点
func NewPeer() (peer *Peer, err error) {
  config := webrtc.Configuration{
    ICEServers: []webrtc.ICEServer{
      webrtc.ICEServer{
        URLs: []string{"stun:stun.l.google.com:19302"},
      },
    },
  }
  conn, err := webrtc.NewPeerConnection(config)
  if err != nil {
    return
  }
  peer = &Peer{
    conn: conn,
    buff: make(chan string, 100),
  }
  log.Println("peer created")
  return
}

// NewLocalPeer 创建本地Peer节点
func NewLocalPeer() (peer *Peer, err error) {
  peer, err = NewPeer()
  if err != nil {
    return
  }
  err = peer.CreateDataChannel()
  return
}

// NewRemotePeer 创建远端Peer节点
func NewRemotePeer() (peer *Peer, err error) {
  peer, err = NewPeer()
  if err != nil {
    return
  }
  peer.OnDataChannel()
  return
}

// CreateDataChannel 创建本地数据通道
func (p *Peer) CreateDataChannel() (err error) {
  data, err := p.conn.CreateDataChannel("data", nil)
  if err != nil {
    return
  }
  p.data = data
  p.data.OnOpen(func() {
    log.Println("local data channel open")
    for {
      text := <-p.buff
      p.data.SendText(text)
    }
  })
  p.data.OnClose(func() {
    log.Println("local data channel close")
  })
  log.Println("local datachannel created")
  return
}

// OnDataChannel 等待远端数据通道
func (p *Peer) OnDataChannel() {
  p.conn.OnDataChannel(func(data *webrtc.DataChannel) {
    p.data = data
    p.data.OnOpen(func() {
      log.Println("remote data channel open")
    })
    p.data.OnClose(func() {
      log.Println("remote data channel close")
    })
    p.data.OnMessage(func(msg webrtc.DataChannelMessage) {
      log.Println("remote on message")
      log.Println(msg)
      if msg.IsString {
        log.Println(string(msg.Data))
      }
    })
    log.Println("remote on datachannel")
  })
}

// Connect 连接两个peer
func (p *Peer) Connect(o *Peer) (err error) {
  offer, err := p.conn.CreateOffer(nil)
  if err != nil {
    return
  }
  if err = p.conn.SetLocalDescription(offer); err != nil {
    return
  }
  if err = o.conn.SetRemoteDescription(offer); err != nil {
    return
  }
  log.Println("local description created")
  answer, err := o.conn.CreateAnswer(nil)
  if err != nil {
    return
  }
  if err = o.conn.SetLocalDescription(answer); err != nil {
    return
  }
  if err = p.conn.SetRemoteDescription(answer); err != nil {
    return
  }
  log.Println("remote description created")
  return
}

// Send 从一端向另一端发送数据
func (p *Peer) Send(text string) {
  p.buff <- text
}

func init() {
  log.SetOutput(os.Stdout)
}

func main() {
  local, err := NewLocalPeer()
  if err != nil {
    log.Fatal(err)
  }
  remote, err := NewRemotePeer()
  if err != nil {
    log.Fatal(err)
  }
  err = local.Connect(remote)
  if err != nil {
    log.Fatal(err)
  }
  local.Send("Hello World")
  select {}
}
```