# 利用winpcap实现网络抓包

在windows下，我们通常是通过fiddler这样的工具进行网络抓包，但其实这些抓包工具底层都是使用了像`winpcap`这样的c库，也就是说，我们也可以通过调用`winpcap`，直接通过编程的手段实现网络抓包。在python中调用`winpcap`，首先我们需要安装`winpcapy`模块：

```shell
pip install winpcapy
```

通过调用`winpcapy`模块下的`WinPcapUtils.capture_on_device_name`方法，我们就可以实现网络抓包。下面这段代码，就是通过python调用`winpcap`库的一个例子：

```python
#!/usr/bin/env python
# coding=utf-8

from winpcapy import *

def packet_callback(win_pcap, param, header, pkt_data):
    try:
        try:
            print pkt_data
        except:
            pass
    except:
        win_pcap.stop()
        sys.exit(0)

def get_device(idx):
    with WinPcapDevices() as devices:
        i = 0
        for device in devices:
            if (i == idx):
                return device
            i += 1
        return None

def main():
    # 通过winpcapy抓包
    WinPcapUtils.capture_on_device_name(get_device(1).name, packet_callback)

if __name__ == '__main__':
    main()
```

但在实际运行的时候，由于`winpcapy`底层调用的是`winpcap`中的`pcap_loop`函数，该函数如果抓不到包，会一直阻塞当前进程，导致没有办法让程序停下来。

为了解决这个问题，只能自己再另外封装一次。翻了一下`winpcapy`的源码，看到他实际是使用了另外一份别人实现的，利用`ctypes`对`libpcap/winpcap`进行包裹的代码，具体请移步[这里](https://code.google.com/archive/p/winpcapy)。这份代码直接把`libpcap/winpcap`的c接口原封不动的暴露了出来，因此调用起来更加随心所欲，一下是直接通过其暴露的c接口实现网络抓包：

```python
#!/usr/bin/env python
# coding=utf-8

from ctypes import *
# https://code.google.com/archive/p/winpcapy/
from winpcapy import *

class Capture(object):
    def __init__(self, descriptor):
        self.descriptor = descriptor

    def __del__(self):
        pcap_close(self.descriptor)

    def filter(self, expression):
        """
            设置pcap的过滤器，首先需要通过pcap_compile对过滤器表达式进行编译
            然后通过pcap_setfilter把编译得到的过滤器设置到pcap中
        """
        fp = bpf_program()

        if pcap_compile(self.descriptor, byref(fp), expression, 1, 0) == -1:
            raise RuntimeError('Error compile filter: %s' % pcap_geterr(self.descriptor))

        if pcap_setfilter(self.descriptor, byref(fp)) == -1:
            raise RuntimeError('Error setting filter: %s' % pcap_geterr(self.descriptor))
        

    def next(self):
        """
            获取pcap中的下一个数据包，这通过调用pcap_next_ex方法
            分别获得包头和数据包，并利用包头中的len字段来截取数据包中的有效数据部分
        """
        header = POINTER(pcap_pkthdr)()
        pkt_data = POINTER(c_ubyte)()

        if pcap_next_ex(self.descriptor, byref(header), byref(pkt_data)) == -1:
            raise RuntimeError('Error reading packets: %' % pcap_geterr(self.descriptor))

        header = header.contents
        pkt_data = string_at(pkt_data, header.len)

        return header, pkt_data

    @staticmethod
    def on_device(device, snaplen, promisc, to_ms):
        """
            通过指定设备名称，截获该设备上的收发的数据
        """
        errbuf = create_string_buffer(PCAP_ERRBUF_SIZE)
        descriptor = pcap_open_live(device, snaplen, promisc, to_ms, errbuf)

        if descriptor == None:
            raise RuntimeError('Unable to open the device: %s' % errbuf.value)

        print('Listening on %s...\n' % device)

        return Capture(descriptor)


class Device(object):
    def __init__(self, name, description):
        self.name = name
        self.description = description

    def open(self, snaplen=65536, promisc=1, to_ms=1000):
        return Capture.on_device(self.name, snaplen, promisc, to_ms)

    @staticmethod
    def list():
        """
            列出当前计算机上所有的网卡设备，在libpcap中
            设备系统是存储在一个pcap_if_t的链表中，所以需要把他转换成一个python的list
        """
        errbuf = create_string_buffer(PCAP_ERRBUF_SIZE)
        alldevs = POINTER(pcap_if_t)()

        if pcap_findalldevs(byref(alldevs), errbuf) == -1:
            raise RuntimeError('Error in pcap_findalldevs: %s' % errbuf.value)

        devs = []
        dev = alldevs.contents

        while dev:
            devs.append(Device(dev.name, dev.description))

            if dev.next:
                dev = dev.next.contents
            else:
                dev = None

        pcap_freealldevs(alldevs)

        return devs

def main():
    devs = Device.list()
    
    for dev in devs:
        print('%s (%s)' % (dev.name, dev.description))

    dev = devs[1]
    # 打开选中的设备
    cap = dev.open()
    # 并且设置只拦截80端口的数据
    cap.filter('port 80')

    while True:
        header, pkt_data = cap.next()

        if pkt_data == '':
            continue

        try:
            print pkt_data
        except:
            pass

if __name__ == '__main__':
    try:
        main()
    except:
        sys.exit(-1)
```

使用以上的代码，我们已经可以把网卡上流过的数据包全部抓取下来，但如果我们直接打印数据包，会发现他们都是一些赤裸裸的二进制数据，为了解析这些二进制数据，当然我们可以参照tcp/ip协议对数据包的定义进行解析，但如果你不希望浪费太多时间在这方面，python也有`dpkt`模块来帮助我们完成相应的工作。首先我们来安装一下`dpkt`模块：

```shell
pip install dpkt
```

然后我们实现一下解析二进制数据的类：

```python
import socket
import sys
import dpkt
from dpkt.compat import compat_ord

class Sniffer(object):
    def __init__(self, pkt_data):
        self.eth = dpkt.ethernet.Ethernet(pkt_data)

        assert isinstance(self.eth.data, dpkt.ip.IP)
        self.ip4 = self.eth.data

        assert isinstance(self.ip4.data, dpkt.tcp.TCP)
        self.tcp = self.ip4.data

        assert self.tcp.dport == 80 or self.tcp.sport == 80
        if self.tcp.dport == 80:
            self.req = dpkt.http.Request(self.tcp.data)
            self.res = None
        else:
            self.req = None
            self.res = dpkt.http.Response(self.tcp.data)

    def __repr__(self):
        eth = self.eth
        ip4 = self.ip4
        tcp = self.tcp
        req = self.req
        res = self.res

        info =  '\n{} -> {}'.format(self.get_mac_addr(eth.src), \
            self.get_mac_addr(eth.dst))
        info += '\n{}:{} -> {}:{}'.format(self.get_ip_addr(ip4.src), tcp.sport, \
            self.get_ip_addr(ip4.dst), tcp.dport)
        info += ' (TTL: {}, LEN: {})'.format(ip4.ttl, ip4.len)

        if req:
            info += '\n{} {} {}'.format(req.version, req.method, req.uri)

            for key, val in req.headers.items():
                info += '\n{}: {}'.format(key, val)

        if res:
            info += '\n{} {} {}'.format(res.version, req.status, req.reason)

            for key, val in res.headers.items():
                info += '\n{}: {}'.format(key, val)

        return info
        
    def get_mac_addr(self, mac_raw):
        return ':'.join(['%02x' % compat_ord(b) for b in mac_raw])

    def get_ip_addr(self, ip_raw):
        return socket.inet_ntoa(ip_raw)
```

现在我们就可以利用这个`Sniffer`类来帮助我们把二进制数据转换成可读性更高的文本内容：

```python
print Sniffer(pkt_data)
```

以上～～