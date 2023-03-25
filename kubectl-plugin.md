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
```go
func NewDefaultKubectlCommandWithArgs(o KubectlOptions) *cobra.Command {
	cmd := NewKubectlCommand(o)

	if o.PluginHandler == nil {
		return cmd
	}

	if len(o.Arguments) > 1 {
		cmdPathPieces := o.Arguments[1:]

		// 命令未找到时，抛出err
		if _, _, err := cmd.Find(cmdPathPieces); err != nil {
			// Also check the commands that will be added by Cobra.
			// These commands are only added once rootCmd.Execute() is called, so we
			// need to check them explicitly here.
			var cmdName string // first "non-flag" arguments
			for _, arg := range cmdPathPieces {
				if !strings.HasPrefix(arg, "-") {
					cmdName = arg
					break
				}
			}

			switch cmdName {
			case "help", cobra.ShellCompRequestCmd, cobra.ShellCompNoDescRequestCmd:
				// Don't search for a plugin
			default:
				// 进入插件处理逻辑
				if err := HandlePluginCommand(o.PluginHandler, cmdPathPieces); err != nil {
					fmt.Fprintf(o.IOStreams.ErrOut, "Error: %v\n", err)
					os.Exit(1)
				}
			}
		}
	}

	return cmd
}
```
```go
// HandlePluginCommand 尝试找到命令行参数匹配的PATH下可执行命令
func HandlePluginCommand(pluginHandler PluginHandler, cmdArgs []string) error {
	var remainingArgs []string // all "non-flag" arguments
	for _, arg := range cmdArgs {
		if strings.HasPrefix(arg, "-") {
			break
		}
		remainingArgs = append(remainingArgs, strings.Replace(arg, "-", "_", -1))
	}

	if len(remainingArgs) == 0 {
		// the length of cmdArgs is at least 1
		return fmt.Errorf("flags cannot be placed before plugin name: %s", cmdArgs[0])
	}

	foundBinaryPath := ""

	// 尝试找到匹配命令行参数最长的可执行命令
	for len(remainingArgs) > 0 {
		path, found := pluginHandler.Lookup(strings.Join(remainingArgs, "-"))
		if !found {
			remainingArgs = remainingArgs[:len(remainingArgs)-1]
			continue
		}

		foundBinaryPath = path
		break
	}

	if len(foundBinaryPath) == 0 {
		return nil
	}

	// 执行找到的二进制文件
	if err := pluginHandler.Execute(foundBinaryPath, cmdArgs[len(remainingArgs):], os.Environ()); err != nil {
		return err
	}

	return nil
}
```
```go
func (h *DefaultPluginHandler) Lookup(filename string) (string, bool) {
	for _, prefix := range h.ValidPrefixes {
		// 找到PATH路径下匹配的可执行文件
		path, err := exec.LookPath(fmt.Sprintf("%s-%s", prefix, filename))
		if shouldSkipOnLookPathErr(err) || len(path) == 0 {
			continue
		}
		return path, true
	}
	return "", false
}
```
# 参考
[用插件扩展 kubectl](https://kubernetes.io/zh-cn/docs/tasks/extend-kubectl/kubectl-plugins/)