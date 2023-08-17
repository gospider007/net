# 版本
```
go 1.21.0
```
# 修改指纹
## 添加函数
```go
func NewClientConn(closeCallBack func(), c net.Conn, h2Ja3Spec ja3.H2Ja3Spec) (*ClientConn, error) {
	var headerTableSize uint32 = 65536
	var maxHeaderListSize uint32 = 262144
	var streamFlow uint32 = 6291456

	if h2Ja3Spec.InitialSetting != nil {
		for _, setting := range h2Ja3Spec.InitialSetting {
			switch setting.Id {
			case 1:
				headerTableSize = setting.Val
			case 6:
				maxHeaderListSize = setting.Val
			case 4:
				streamFlow = setting.Val
			}
		}
	} else {
		h2Ja3Spec.InitialSetting = []ja3.Setting{
			{Id: 1, Val: headerTableSize},
			{Id: 2, Val: 0},
			{Id: 3, Val: 1000},
			{Id: 4, Val: 6291456},
			{Id: 6, Val: maxHeaderListSize},
		}
	}
	if !h2Ja3Spec.Priority.Exclusive && h2Ja3Spec.Priority.StreamDep == 0 && h2Ja3Spec.Priority.Weight == 0 {
		h2Ja3Spec.Priority = ja3.Priority{
			Exclusive: true,
			StreamDep: 0,
			Weight:    255,
		}
	}
	if h2Ja3Spec.OrderHeaders == nil {
		h2Ja3Spec.OrderHeaders = []string{":method", ":authority", ":scheme", ":path"}
	}
	if h2Ja3Spec.ConnFlow == 0 {
		h2Ja3Spec.ConnFlow = 15663105
	}
	//开始创建客户端
	return (&Transport{
		closeCallBack:             closeCallBack,
		h2Ja3Spec:                 h2Ja3Spec,
		streamFlow:                streamFlow,
		MaxDecoderHeaderTableSize: headerTableSize,   //1:initialHeaderTableSize,65536
		MaxEncoderHeaderTableSize: headerTableSize,   //1:initialHeaderTableSize,65536
		MaxHeaderListSize:         maxHeaderListSize, //6:MaxHeaderListSize,262144
	}).NewClientConn(c)
}
```

## 修改标头顺序
### 修改 enumerateHeaders 函数中的请求头顺序: 函数传参f 改为f2 ,函数内部重新定义函数f
```go
	//f 改为f2
	func(f2 func(name, value string)) {
		//开头
		headers := http.Header{}
		f := func(name, value string) {
			headers.Add(name, value)
		}
		//中间代码不变
		//中间代码不变
		//中间代码不变
		//中间代码不变

		//结尾
		ll := kinds.NewSet[string]()
		for _, kk := range cc.t.h2Ja3Spec.OrderHeaders {
			for i := 0; i < 2; i++ {
				if i == 1 {
					kk = strings.Title(kk)
				}
				if vvs, ok := headers[kk]; ok {
					ll.Add(kk)
					for _, vv := range vvs {
						f2(kk, vv)
					}
					break
				}
			}
		}
		for kk, vvs := range headers {
			for _, vv := range vvs {
				if !ll.Has(kk) {
					f2(kk, vv)
				}
			}
		}
	}
```
## 修改 streamFlow 值,删除 常量 transportDefaultStreamFlow ,并替换为 Transport 中的 streamFlow

## 修改 newClientConn 函数中的   initialSettings 和 WriteWindowUpdate
### 删除常量 transportDefaultConnFlow ， transportDefaultConnFlow  替换为  t.h2Ja3Spec.ConnFlow
### initialSettings 重新赋值
```go
	initialSettings := make([]Setting, len(t.h2Ja3Spec.InitialSetting))
	for i, setting := range t.h2Ja3Spec.InitialSetting {
		initialSettings[i] = Setting{ID: SettingID(setting.Id), Val: setting.Val}
	}
```
## 修改 ClientConn 的 writeHeaders 函数 的 first==true 时候的WriteHeaders 参数增加 Priority 值
```go
		if first {
			cc.fr.WriteHeaders(HeadersFrameParam{//这里是一个指纹检测点
				StreamID:      streamID,
				BlockFragment: chunk,
				EndStream:     endStream,
				EndHeaders:    endHeaders,
				Priority: PriorityParam{
					StreamDep: cc.t.h2Ja3Spec.Priority.StreamDep,
					Exclusive: cc.t.h2Ja3Spec.Priority.Exclusive,
					Weight:    cc.t.h2Ja3Spec.Priority.Weight,
				},
			})
			first = false
		} else {
			cc.fr.WriteContinuation(streamID, endHeaders, chunk)
		}
```
# 添加ctx 以解决http2 代理转发无法及时同步导致的bug
## Server 添加 CloseCallBack 方法  CloseCallBack func() bool
## 修改 serverConn 的  serve() 方法中的代码，将下面代码放到for 循环末尾
```go
	for {
			//源代码
			//下面代码放到末尾
		if sc.srv.CloseCallBack != nil && !sc.inFrameScheduleLoop && !sc.inGoAway && !sc.needToSendGoAway && !sc.needToSendSettingsAck && !sc.needsFrameFlush && !sc.writingFrame && sc.srv.CloseCallBack() {
			return
		}
	}
```
## 修改 closeConn() 函数 用来通知连接的健康，添加这行代码
```go
	if cc.t.closeCallBack != nil {
		defer cc.t.closeCallBack()
	}
```
