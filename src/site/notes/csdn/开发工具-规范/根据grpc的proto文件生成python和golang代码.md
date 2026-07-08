---
{"dg-publish":true,"permalink":"/csdn/开发工具-规范/根据grpc的proto文件生成python和golang代码/","title":"根据grpc的proto文件生成python和golang代码","tags":["python","golang","rpc"],"dg-note-properties":{"category":"开发工具 / 规范","title":"根据grpc的proto文件生成python和golang代码","source":"csdn","created":"2022-06-29","tags":["python","golang","rpc"],"url":"https://blog.csdn.net/weixin_45536921/article/details/125517417"}}
---


python grpc


```
pip install grpcio
pip install grpcio-tools
python3 -m grpc_tools.protoc -I./ --python_out=. --grpc_python_out=. ./helloworld.proto

```
golang grpc


```
go install google.golang.org/protobuf/cmd/protoc-gen-go@v1.28
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@v1.2

export PATH="$PATH:$(go env GOPATH)/bin"

protoc --go_out=. --go_opt=paths=source_relative \
    ./helloworld.proto

```
详见```
https://grpc.io/docs/
```