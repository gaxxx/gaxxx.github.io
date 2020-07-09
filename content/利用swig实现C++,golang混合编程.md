+++
title = "利用swig实现C++,golang混合编程"
date = 2014-06-20
[taxonomies]
tags = ["golang", "swig"]
+++

实验项目：
---------

https://github.com/gaxxx/ctp/

其中实现了两个主要的功能
1. 在golang中调用C++函数
2. C++中触发回调时，调用golang的代码

<!-- more -->

准备swig文件
------------
以下的部分实现了功能1

```
%module ctp %{
  #include "ThostFtdcMdApi.h"
  #include "ThostFtdcTraderApi.h"
  #include "ThostFtdcUserApiDataType.h"
  #include "ThostFtdcUserApiStruct.h"
  #include "helper.h"
%}

%include "../c++/ThostFtdcMdApi.h"
%include "../c++/ThostFtdcTraderApi.h"
%include "../c++/ThostFtdcUserApiDataType.h"
%include "../c++/ThostFtdcUserApiStruct.h"
%include "../helper/helper.h"
```

但是要完成回调，需要用到director的特性
如果要对CThostFtdcTraderSpi的类进行回调，需要定义一个实现了CThostFtdcTraderSpi的golang结构体,通过NewDirector<类名> 进行封装，传给C++.

```
%module(directors="1") CThostFtdcTraderSpi
%feature("director") CThostFtdcTraderSpi;

%insert(go_wrapper) %{

type GoThostFtdcTraderSpi struct {
        CThostFtdcTraderSpi
}

func GTrader(impl CThostFtdcTraderSpi) CThostFtdcTraderSpi {
        return NewDirectorCThostFtdcTraderSpi(&GoThostFtdcTraderSpi{impl})
}
```

在实际的类实现中，需要完成 CThostFtdcTraderSpi的接口实现
为简单起见，我封装了实现所有空接口的变量ThostFtdcTraderSpiImplBase.
因此在编写实际类实现时,只需要重写必要的回调函数接口就可以了

```
type TradeApi struct {
    	ctp.ThostFtdcTraderSpiImplBase
}

//回调函数
func (g *TradeApi) OnFrontConnected() {
        fmt.Printf("connected\n")
}
```

需要注意的是，因为swig利用cgo，因此，不能再引入其他利用cgo编译的库了 :(
如果有其他需要用C/C++实现的函数，可以在 helper.h / helper.cpp 中完成


执行编译

```

#生成ctp_wrap.cxx ctp_gc.c ctp.go
swig -go -c++ -intgosize 64 -soname libctp.so ./ctp.swig
#编译libctp.so
g++ ctp_wrap.cxx -o ctp_wrap.o
g++ helper.cpp -o helper.o
g++ -shared -o libctp.so ctp_wrap.o helper.o
#编译golang package
go tool 6c -I/usr/local/go/pkg/linux_amd64 -D _64BIT ctp_gc.c
go tool 6g ctp.go
go tool pack grc ctp.a ctp.6 ctp_gc.6

```

将libctp.so放到lib路径中 (/usr/lib/)
将ctp.a 放到golang的package中 (/usr/local/go/pkg/linux_amd64/)
将cgp.go放到golang的source中 (/usr/local/go/src/pkt/ctp/)


实际应用

```

//继承ctp.ThostFtdcTraderSpiImplBase
type TradeApi struct {
     ctp.ThostFtdcTraderSpiImplBase
}

//重写回调接口
func (g *TradeApi) OnFrontConnected() {
	fmt.Printf("connected\n")
}

func main() {
	api = ctp.CThostFtdcTraderApiCreateFtdcTraderApi()
	//注册director类
	api.RegisterSpi(ctp.GTrader(&TradeApi{}))
	api.RegisterFront(front)
	api.Init()
	api.Join()
}
```






