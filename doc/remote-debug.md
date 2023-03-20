# 使用Goland远程调试kubernetes源码
## 1. 环境准备
### 1.1 安装go
[golang官网下载](https://golang.google.cn/dl/)
### 1.2 安装goland
[goland官网下载](https://www.jetbrains.com/go/download/#section=mac)
### 1.3 kubernetes源码下载
本机、远程主机各一份，注意代码路径需一致
```git
git clone https://github.com/kubernetes/kubernetes.git
```
## 2. 远程主机配置
### 2.1 安装minikube并启动
[minikube安装启动](https://minikube.sigs.k8s.io/docs/start/)
### 2.2 安装delve
[安装delve](https://github.com/go-delve/delve/tree/master/Documentation/installation)
```shell
git clone https://github.com/go-delve/delve
cd delve
go install github.com/go-delve/delve/cmd/dlv@latest
```
### 2.3 启动
以调试kubectl为例

构建可执行文件，进入kuberctl目录
```shell
cd kubernetes/cmd/kubectl
```
构建
```shell
go build -gcflags="all=-N -l" -o kubectl
```
启动
```shell
dlv exec ./kubectl --listen=:2345 --headless=true --api-version=2 -- get pod
```
至此，远程主机配置已准备完毕
## 3. 本机
[使用goland调试已运行的进程](https://www.jetbrains.com/help/go/attach-to-running-go-processes-with-debugger.html)
### 3.1 创建goland远程调试配置
- Run | Edit Configurations
- Add Go Remote
- Host填写远程主机ip
- Port填写远程主机端口
- Run | Debug 刚刚设置好的配置
即可调试远程主机进程了，当然记得打断点哦
## 参考
[源码讲解](https://developer.ibm.com/articles/a-tour-of-the-kubernetes-source-code/)

[源码目录介绍](https://arunprasad86.medium.com/want-to-understand-kubernetes-source-code-this-is-how-you-can-start-exploring-6eea25e50a69)

[使用Gland调试kubernetes](https://xmudrii.com/posts/debugging-kubernetes/)