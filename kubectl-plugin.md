# kubectl插件
# 1.背景
需要自定义kubectl命令时，可使用kebectl插件扩展命令
# 2.如何使用
## 2.1安装插件
- 可通过kubectl 插件管理器[krew](https://github.com/kubernetes-sigs/krew/)
- 任何其他人制作的可执行文件，放在path目录即可
## 2.2可用的插件列表
```shell
kubectl plugin list
```
## 2.3使用插件
- kubectl优先匹配长命令
- kubectl使用path目录中靠前的同名插件
```shell
kubectl plugin list 
The following compatible plugins are available:

/usr/local/bin/kubectl-convert
```
以kubectl-convert为例
```shell
kubectl convert --help
```
## 2.4制作插件
- 不限制任何语言
- Go语言，可以利用[cli-runtime](https://github.com/kubernetes/cli-runtime)工具库，可参考[CLI 插件示例](https://github.com/kubernetes/sample-cli-plugin)项目
- 可以使用[krew](https://github.com/kubernetes-sigs/krew/)发布插件
# 3.插件源码走读

# 参考
[用插件扩展 kubectl](https://kubernetes.io/zh-cn/docs/tasks/extend-kubectl/kubectl-plugins/)