+++
title = "Mixed C++ and Go Programming with SWIG"
date = 2014-06-20
[taxonomies]
tags = ["golang", "swig"]
+++

Experiment project:
---------

https://github.com/gaxxx/ctp/

This implements two main features:
1. Calling C++ functions from Go
2. Calling Go code from C++ callbacks

<!-- more -->

Preparing the SWIG file
------------
The following implements feature 1:

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

To implement callbacks, you need the director feature. To callback on the CThostFtdcTraderSpi class, define a Go struct that implements CThostFtdcTraderSpi, wrap it with NewDirector<ClassName>, and pass it to C++.

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

In the actual implementation, you need to implement the CThostFtdcTraderSpi interface. For simplicity, I wrapped all empty interface implementations in ThostFtdcTraderSpiImplBase. So when writing the actual implementation, you only need to override the necessary callback functions:

```
type TradeApi struct {
    	ctp.ThostFtdcTraderSpiImplBase
}

//Callback function
func (g *TradeApi) OnFrontConnected() {
        fmt.Printf("connected\n")
}
```

Note that since SWIG uses cgo, you cannot import other libraries that also use cgo :(
If you need other C/C++ functions, implement them in helper.h / helper.cpp.


Building:

```

#Generate ctp_wrap.cxx ctp_gc.c ctp.go
swig -go -c++ -intgosize 64 -soname libctp.so ./ctp.swig
#Build libctp.so
g++ ctp_wrap.cxx -o ctp_wrap.o
g++ helper.cpp -o helper.o
g++ -shared -o libctp.so ctp_wrap.o helper.o
#Build golang package
go tool 6c -I/usr/local/go/pkg/linux_amd64 -D _64BIT ctp_gc.c
go tool 6g ctp.go
go tool pack grc ctp.a ctp.6 ctp_gc.6

```

Place libctp.so in the lib path (/usr/lib/)
Place ctp.a in the Go package path (/usr/local/go/pkg/linux_amd64/)
Place ctp.go in the Go source path (/usr/local/go/src/pkt/ctp/)


Practical usage:

```

//Inherit ctp.ThostFtdcTraderSpiImplBase
type TradeApi struct {
     ctp.ThostFtdcTraderSpiImplBase
}

//Override callback interface
func (g *TradeApi) OnFrontConnected() {
	fmt.Printf("connected\n")
}

func main() {
	api = ctp.CThostFtdcTraderApiCreateFtdcTraderApi()
	//Register director class
	api.RegisterSpi(ctp.GTrader(&TradeApi{}))
	api.RegisterFront(front)
	api.Init()
	api.Join()
}
```
