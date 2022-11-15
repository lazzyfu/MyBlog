- [工具](#工具)
- [安装golang](#安装golang)
- [下载软件包](#下载软件包)
- [生成模板](#生成模板)
- [启动调试](#启动调试)
- [写代码](#写代码)
- [使用orchestrator-client](#使用orchestrator-client)
- [打包](#打包)

## 工具
- vscode
- MAC

## 安装golang
```bash
brew install go@1.16
```

## 下载软件包
```bash
# 安装依赖包
brew install gnu-sed jq

# Clone项目
git clone https://github.com/openark/orchestrator.git orchestrator

# 安装包
cd orchestrator
go mod tidy
```

## 生成模板
```bash
cp conf/orchestrator-sample.conf.json conf/orchestrator.conf.json
```

## 启动调试
```bash
go run go/cmd/orchestrator/main.go -config conf/orchestrator.conf.json -debug http
```

## 写代码
您可以愉快的coding了

## 使用orchestrator-client
脚本位于: resources/bin/orchestrator-client

## 打包
> 打包后的二进制文件位于`./bin/orchestrator`

```bash
env GOOS=linux GOARCH=amd64 go build -i -o ./bin/orchestrator ./go/cmd/orchestrator/main.go
```