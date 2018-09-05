# 用Golang进行数据封装生成可供C/C++调用SO库的SDK开发

游戏服务端研发部门要用C/C++开发，由于框架限制等原因不能直接使用socket来和我们的聊天系统进行数据交换，所以需要我这边模拟接收发送数据行为并生成一个so库供C/C++直接调用方法。当时考虑到直接用C/C++写的话开发成本有点高而且我也不熟悉C++网络编程，网上看到Golang程序可以和C/C++互相调用，就决定用Golang来处理了（因为Golang的网络编程和Json处理等机制比较方便）。

现在已经把SDK交付完成，期间遇到很多问题也踩过很多次坑，最终还是解决了，在这里总结一下。

## 前提知识

- golang基本语法，花了一晚上加上上午一小时左右看完，但复杂的机制如gc，多线程goroutine，channel，同步锁互斥等还需要具体用到时再详细去研究。
- golang和c的转化编译命令操作。
- 多线程知识，线程间通信等。
- 设置回调
- TCP基础知识，粘包处理。
- 和聊天系统服务器的中转协议。

## 架构

SDK一共分成三个文件:

1. sdk.go
2. util.go
3. log.go

实际上是同一个包，包名都为main，因此可以分不同的文件来写，go在编译时会把同一个包的所有文件都编译成一个文件。

### 思路

主线程让研发部门调用发送或初始化方法，调用连接方法后，如果连接成功了就开启一个子线程来接收数据包，并以回调的方式返回给研发部门，同时可以一直在主线程调用发送方法。

## 实际问题

只是个人的写法和理解，可能还会有更好的处理。

### TCP粘包处理以及json转换

#### json转换的问题。

发送和接收数据包都需要处理。

和动态语言不一样，golang没有对象类型并且无法动态添加删除字段，比较常用的是struct和map来和json进行互换（map可以）。在中转协议中自定义了包结构和信息结构，因此我使用了struct，需要在一开始就声明字段。

```
// 消息结构
type Cb struct {
	Code string      `json:"code"`
	Data interface{} `json:"data"`
}
```

golang提供一个标准库`encoding/json`把struct和map来和json进行互换，所以struct->json可以用标准库的Marshal函数，但json->struct要并获取一些字段的值时用标准库就比较复杂，因此我们可以用`github.com/tidwall/gjson`库。以传入一个json格式的string为例，我要获取该string里的字段值并重新封装成json。

```
func kf_send(str string) int {
	if !client.isInitServer {
		if isSetLog {
			Error.Println("unregister to VIPKf Server.Please Call init_server first!")
		}
		return 2
	}
	code := gjson.Get(str, "code").String()
	data := gjson.Get(str, "data").Value()
	cb := &Cb{
		code,
		data,
	}
	pack, err := json.Marshal(cb)
	if err != nil {
		if isSetLog {
			Error.Println("Format:", err.Error())
		}

		return 5
	}
	return _send(pack)
}
```

#### TCP粘包处理

数据包的格式是我们自己定义的，一个比较简单的没校验的格式，只有32 bit的头部来存放数据包长度，后面就是数据包。

```
//TCP数据包
type TcpPackage struct {
	Length int32  //数据长度
	Msg    []byte //数据
}
```

##### 发送

数据包的粘包处理，首先把定义一个封包的Pack方法

```
// 封包,TCP粘包处理
func (p *TcpPackage) Pack(writer io.Writer) error {
	var err error
	err = binary.Write(writer, binary.LittleEndian, &p.Length)
	err = binary.Write(writer, binary.LittleEndian, &p.Msg)

	return err
}
```

这里统一发送数据：

```
// 统一处理格式后发送
func _send(pack []byte) int {
	//打包
	sendPackage := &TcpPackage{
		Length: int32(len(pack)),
		Msg:    pack,
	}
	buf := bytes.NewBuffer(make([]byte, 0, sendPackage.Length))
	err := sendPackage.Pack(buf)
	if err != nil {
		if isSetLog {
			Error.Println("Pack Error:", err.Error())
		}
		return 5
	}
	length, err := client.connection.Write(buf.Bytes())
	if err != nil {
		if isSetLog {
			Error.Println("send Error:", err.Error(), length)
		}
		return 4
	}
	return 0
}
```

##### 接收

定义一个解包方法：

```
// 解包
func (p *TcpPackage) Unpack(reader io.Reader) error {
	var err error
	err = binary.Read(reader, binary.LittleEndian, &p.Length)
	p.Msg = make([]byte, p.Length)
	err = binary.Read(reader, binary.LittleEndian, &p.Msg)
	return err
}
```

接收数据，需要把接收到小于4 byte的包去掉以及把包长度超过一次接收到长度的包拼接起来。

```
scanner := bufio.NewScanner(conn)
scanner.Split(func(data []byte, atEOF bool) (advance int, token []byte, err error) {
	if !atEOF {
		if len(data) > 4 {
			length := int32(0)
			binary.Read(bytes.NewReader(data[:4]), binary.LittleEndian, &length)
			if int(length)+4 <= len(data) {
				return int(length) + 4, data[:int(length)+4], nil
			}
		}
	}
	return
})
//读取数据
for scanner.Scan() {
	scannedPack := new(TcpPackage)
	scannedPack.Unpack(bytes.NewReader(scanner.Bytes()))
	
	packStr := string(scannedPack.Msg)
	if isSetLog && isDebug {
		Debug.Println("[", GoID(), "]", "receive packet:", packStr)
	}

	code := gjson.Get(packStr, "code").String()
	data := gjson.Get(packStr, "data")

	// 判断是否已经初始化服务器
	if "init_server_response" == code {
		ret := data.Get("ret").Int()
		if 0 == ret {
			client.isInitServer = true
			GoSysCallback(100002, "register kfserver OK")
			continue
		}
	}
	GoReportData(code, data.String())
}
```

### 设置回调

作为一个TCP客户端，我们想什么时候发送数据是由研发决定的，直接调用相应的方法即可。但同时我们需要一直监听接收数据，在Golang中当然可以开一个goroutine来一直`client.Read()`，但问题是每次我们收到数据后怎么让C那边收到通知并接收到数据呢？

- 方法一：每次接收到数据包后放到一个消息队列里，C那边写一个监听来读取每次的数据包。这个方法明显开发难度都在研发部门那边，并且设计得不是很友好。
- 方法二：C注册一个回调函数，每次有数据过来时进行处理。这方法非常优雅，C那边的开发难度也非常低，因此才比较符合我们平台技术支持中心的定位嘛。

因为我们是要生成一个.so和一个.h文件，不能直接给研发部门设置回调函数，所以我们要写一个中介文件bridge.c来注册回调函数。

```
/*
Author: ChenYaoming(0816)
Time: 2018-09-05
Description: callback bridge
*/
#include "_cgo_export.h"

int KfReceiveMsg(ptfFuncReceiveMsg pfmsg,const char* code,const char* data){
    if (NULL == pfmsg){
        return -1;
    }
    if (NULL == code){
        return -1;
    }
    if (NULL == data){
        return -1;
    }
    return pfmsg(code, data);
}

int KfSysCallback(ptfFuncSysCallback pfsys,int ret,const char* data){
    if (NULL == pfsys){
        return -1;
    }
    return pfsys(ret,data);
}
```

这里是两个回调，第一个接收`int, char*`，第二个接收`char*, char*`，是我逻辑需要，其实格式是一样的，看一个就行。

回到sdk.go这边，我们暴露给研发一个`kf_set_callback`函数，让他们在这里设置回调函数的名字。

```
/*

#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef int (*ptfFuncReceiveMsg)(const char* code,const char* data);
typedef int (*ptfFuncSysCallback)(int ret,const char* msg);
extern int KfReceiveMsg(ptfFuncReceiveMsg pfmsg,const char* code,const char* data);
extern int KfSysCallback(ptfFuncSysCallback pfsys,int ret,const char* msg);

*/
import "C"
// 设置回调函数，参数一：SDK系统回调函数 参数二：客服消息回调函数
//export kf_set_callback
func kf_set_callback(sysCallback C.ptfFuncSysCallback, receiveMsg C.ptfFuncReceiveMsg) {
	if isSetLog && isDebug {
		Debug.Println("[", GoID(), "]", "SetCallBack ok")
	}
	isSetCallbackBefore = true
	sysCallbackFunc = sysCallback
	callBackFunc = receiveMsg
}
```

注意，go和c之间的转换必须要引入"C"模块，而`import "C"`上面的注释（注解）是设置回调时必需的，会被编译器解析编译。

然后处理回调的格式，这里是贴出一个回调处理，另一个给出定义。

```
//系统回调
func GoSysCallback(ret int, msg string) {
	if len(msg) == 0 {
		if isSetLog && isDebug {
			Debug.Println("[", GoID(), "]", "Callback To Game:", msg)
		}
	}
	bdata := []byte(msg)
	if bdata == nil {
		if isSetLog {
			Error.Println("bdata is nil")
		}
		return
	}
	C.KfSysCallback(sysCallbackFunc, C.int(ret), (*C.char)(unsafe.Pointer(&bdata[0])))
}
// 回调函数中转
func GoReportData(code string, data string) {
    ...
}
```

这样的话，我们在上面的接收数据包代码中也看到，接收到处理好的数据后，我们可以直接用`GoReportData(code string, data string)`来触发。

而研发的C代码中就能非常方便地使用返回的code和data了。一个demo.c是：

```
/*
 * 客服消息回调
 * param code 协议号
 * param data 消息内容
 */
int OnReceiveMsg(const char *code, const char *data)
{
    //printf("OnReceiveMsg %s %s\r\n", code, data);
    
    //客服上线
    if(strcmp(code,"register_kf")){
        //TODO todo something
    }
    return 0;
}
```

### 错误处理以及日志打印

#### 错误/异常

Golang的错误/异常处理毁誉参半（或者毁的更多一点），个人理解可以做成一个链式错误接收并返回的机制。例如我曾经在代码中一个函数调用链是：

`kf_send_msg_to_kf(str string) int -> kf_send_data_by_code(code string, data string) error -> _send(pack []byte) error`

每次调用都有一个错误接收，如果其中任何一个环节出错就可以

`return errors.New("this func name" + err)`

上层函数接收到错误就能根据这个错误链来找到错误调用，选择处理还是继续向上层返回错误。这样对找到错误非常方便。

#### 日志

我开了一个debug文件记录debug信息，一个error文件来记录错误信息，golang的日志系统其实已经比较方便了，只需要配置一下logger即可。

```
errLogFile, err := os.OpenFile(absolutePath+"errors_"+time.Now().Format("2006_01_02")+".log",
	os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0666)
if err != nil {
	log.Println("打开error日志文件失败：", err)
	return 1
}
Error = log.New(io.Writer(errLogFile), "Error:",
		log.Ldate|log.Ltime|log.Lshortfile)
```

然后使用`Error.PrintLn()`就可以把日志打印到该文件里。

### 状态检测和自动重连

这是我花了最长时间搞的东西，最开始的写法会造成线程间嵌套创建，父进程无法被释放导致数据量一大就内存泄漏。用pprof和写了几个自动运行脚本来查看性能，发现内存会一直上升，gc没用吗？又去查了很多goalng里gc相关的知识，知道了变量逃逸、不同版本gc效果差异等等内容，最后还是得出结论：自己的写法错了。

于是下面是重写后正常的版本。初始化时开一个线程阻塞监听重连信号：

```
var reconnEvent chan int = make(chan int)
//进程检查网络连接
func checkCon() {
	if isSetLog && isDebug {
		Debug.Println("[", GoID(), "]", "checkCon runing....")
	}
	for {
		if _closed {
			break
		}
		//状态为0时代表需要重连
		status := <-reconnEvent
		if status == 0 {
			if isSetLog && isDebug {
				Debug.Println("[", GoID(), "]", "reConnecting now...", SERVER_ADDR)
			}
			//进行连接
			go connect()
		} else if status == -1 { //主动关闭
			if isSetLog && isDebug {
				Debug.Println("[", GoID(), "]", "VIPKf close now")
			}
			//标识已主动关闭
			_closed = true
			break
		}
		//延时，避免卡死
		time.Sleep(PERIOD_TIME * time.Second)
	}
}
```

当channel收到0信号时说明需要重连，这时另开一个线程来进行重连。需要重连是啥时候呢？发送或接收数据时出错，就`reconnEvent <- 0`通知重连。如果是接收数据时出错，还会退出该线程。

新开的线程中会尝试去重连，如果成功就阻塞接收数据（recv_pack()），否则再次给channel发出信号通知重连，并退出当前线程。

```
func connect() {
	var err error
	conn, err = net.DialTimeout("tcp", SERVER_ADDR, TIMEOUT*time.Second)
	defer func() {
		_closing = true
		if conn != nil {
			if isSetLog {
				Error.Println("Connect closed now")
			}
			conn.Close()
		}
		if client.connection != nil {
			if isSetLog {
				Error.Println("Close client connection")
			}
			client.connection.Close()
		}
		client.connection = nil
		client.isInitServer = false
		GoSysCallback(100001, "VIPKF Server lost")
		reconnEvent <- 0
	}()
	if err != nil {
		if isSetLog {
			Error.Println("Connect Error:", err.Error())
		}
		return
	}
	err = conn.(*net.TCPConn).SetKeepAlive(true)
	err = conn.(*net.TCPConn).SetKeepAlivePeriod(KeepAlivePeriod * time.Second)

	//绑定连接
	client.connection = conn
	//设状态为已连接
	_closing = false
	//回调给SDK
	GoSysCallback(100000, "VIPKF Server Connected")
	//接收消息
	recv_pack(conn)
}
```

### 编译生成.so和.h

用到cgo的内容。不过简单来说，就是编译时运行

`go build -x -v -ldflags "-s -w" -buildmode=c-shared -o libClient.so .`

之所以用.是因为我当前文件夹中有

- sdk.go
- util.go
- log.go
- bridge.go

要一起编译才能生成相应的sdk。这时，C那边就能方便地直接使用了。



## 参考资料

[Golang生成共享库(shared library)以及Golang生成C可调用的动态库.so和静态库.a](https://blog.csdn.net/linuxandroidwince/article/details/78723441)

[Scanner](https://zhuanlan.zhihu.com/p/37673679)

[Golang学习](https://docs.hacknode.org/gopl-zh/ch5/ch5-09.html)