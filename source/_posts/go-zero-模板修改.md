---
title: go-zero 快速开发
layout: post
tags:
  - go-zero
categories:
  - go
date: 2022-06-22 10:32:33
---



### go-zero 快速开发

项目地址 https://github.com/zeromicro/go-zero.git

这个微服务框架本身自带一个goctl的命令行工具，用于一部分代码生成，但是实际使用上来说，在api调用rpc服务的时候，一旦rpc的出参和入参数量太多，就会是一个相对麻烦的事情，所以还需要修改一下它自带的代码生成模板。

##### goctl 版本1.3.5

初始化模板命令，默认是没有这个文件夹的

```
goctl template init
goctl env
GOCTL_HOME=C:\Users\56414\.goctl
```

在 GOCTL_HOME 目录修改 handler.tpl 文件

```go
package {{.PkgName}}

import (
	"encoding/json"
	"net/http"
	"github.com/zeromicro/go-zero/core/logx"
	"github.com/zeromicro/go-zero/rest/httpx"
	{{.ImportPackages}}
)

func {{.HandlerName}}(svcCtx *svc.ServiceContext) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		type ErrorReply struct {
			Status  int    `json:"status"`
			Message string `json:"message"`
		}
		{{if .HasRequest}}var req types.{{.RequestType}}
		if err := httpx.Parse(r, &req); err != nil {
			w.WriteHeader(http.StatusOK)
            w.Header().Set("Content-Type", "application/json")
			res, _ := json.Marshal(&ErrorReply{Status: 400, Message: err.Error()})
			w.Write(res)
			return
		}

		{{end}}l := {{.LogicName}}.New{{.LogicType}}(r.Context(), svcCtx)
		{{if .HasResp}}resp, {{end}}err := l.{{.Call}}({{if .HasRequest}}&req{{end}})
		if err != nil {
			w.WriteHeader(http.StatusOK)
			w.Header().Set("Content-Type", "application/json")
			res, _ := json.Marshal(&ErrorReply{Status: 400, Message: err.Error()})
			w.Write(res)
		} else {
			{{if .HasResp}}
				w.Header().Set("Content-Type", "application/json")
				w.WriteHeader(http.StatusOK)
				if n, err := w.Write(resp); err != nil {
					// http.ErrHandlerTimeout has been handled by http.TimeoutHandler,
					// so it's ignored here.
					if err != http.ErrHandlerTimeout {
						logx.Errorf("write response failed, error: %s", err)
					}
				} else if n < len(resp) {
					logx.Errorf("actual bytes: %d, written bytes: %d", len(resp), n)
				}				
			{{else}}httpx.Ok(w){{end}}
		}
	}
}
```

在 GOCTL_HOME 目录修改 logic.tpl 文件

```go
package {{.pkgName}}

import (
	{{.imports}}
	"github.com/go-playground/validator/v10"
	"github.com/jinzhu/copier"
	"encoding/json"
)

type {{.logic}} struct {
	logx.Logger
	ctx    context.Context
	svcCtx *svc.ServiceContext
}

func New{{.logic}}(ctx context.Context, svcCtx *svc.ServiceContext) *{{.logic}} {
	return &{{.logic}}{
		Logger: logx.WithContext(ctx),
		ctx:    ctx,
		svcCtx: svcCtx,
	}
}

func (l *{{.logic}}) {{.function}}({{.request}}) {{.responseType}} {
	validate := validator.New()
	if err := validate.Struct(req); err != nil {
		return []byte{}, err
	}

	bpReq := TODOREQ.{{.function}}Req{}
	if err := copier.Copy(&bpReq, req); err != nil {
		return []byte{}, err
	}

	res, err := l.svcCtx.TODO_SERVER.{{.function}}(l.ctx, &bpReq)
	if err != nil {
		return []byte{}, err
	}

	res_json, err := json.Marshal(res)
	if err != nil {
		return []byte{}, err
	}

	return res_json, nil
}
```

返回值类型统一为 []byte

api定义参考

```go
	@doc(
		summary: "退出登录"
	)
	@handler Logout2Handler
	post /hospital/logout2(LogoutReq) returns ([]byte);
```

代码实现

```go
func (l *Logout2Logic) Logout2(req *types.LogoutReq) (resp []byte, err error) {
	validate := validator.New()
	if err := validate.Struct(req); err != nil {
		return []byte{}, err
	}

	bpReq := user.LogoutReq{}
	if err := copier.Copy(&bpReq, req); err != nil {
		return []byte{}, err
	}

	res, err := l.svcCtx.UserRulesServer.Logout(l.ctx, &bpReq)
	if err != nil {
		return []byte{}, err
	}

	res_json, err := json.Marshal(res)
	if err != nil {
		return []byte{}, err
	}

	return res_json, nil
}
```

到这里还有一个问题，rpc 生成的结构体转 json 有些空值的字段是会被丢弃的，为了避免这个问题，

所有还需要修改 protobuf-go 的源码。

```go
git clone https://github.com/SpadesA99/protobuf-go.git
./build.bat
```

然后将 protoc-gen-go.exe 复制到 go bin目录
