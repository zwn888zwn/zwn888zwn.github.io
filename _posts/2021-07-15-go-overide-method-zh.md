---
layout    : post
title     : "go修改grpc-gateway HTTP响应体的ContentType"
date      : 2021-07-15
lastupdate: 2020-07-15
categories: go
---

  修改grpc-gateway HTTP响应体的ContentType

通过源码可以看出，http响应头部通过marshaler设置。原有marshaler使用的是jsonb，需要重写marshaler的ContentType()函数。![](/assets/img/2021-07-15-go-overide-method-zh/16663181847793.jpg)

## Go重写方法

期望重写JSONPb的ContentType()方法，修改其返回值

```go
type JSONPb jsonpb.Marshaler

// ContentType always returns "application/json".
func (*JSONPb) ContentType() string {
	return "application/json"
}
```

用struct封装后重写ContentType()函数

```go
type CustomJSONPb struct {
	runtime.JSONPb
}
func (*CustomJSONPb) ContentType() string {
	return "application/json; charset=utf-8"
}
```

传入自定义CustomJSONPb结构体

```go
func APIHandler() http.Handler {
	customJSONPb := CustomJSONPb{}
	customJSONPb.OrigName = true
	customJSONPb.EmitDefaults = true
	gwMux := runtime.NewServeMux(runtime.WithMarshalerOption(runtime.MIMEWildcard, &customJSONPb))
	//mux register error handle func
	runtime.WithProtoErrorHandler(errorHandler)(gwMux)
	_ = registerAPIFunctions(gwMux)

	mux := http.NewServeMux()
	mux.Handle("/", gwMux)

	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		mux.ServeHTTP(w, r)
	})
}
```

