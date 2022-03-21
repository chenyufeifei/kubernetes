## api-server

### 参数

api server 的运行参数存储在 `ServerRunOption` 结构体中





### 启动

Kube-apiserver 提供了三个服务：APIExtensionsServer，kubeAPIServer、AggregatorServer

创建三个服务，

```go
func CreateServerChain(completedOptions completedServerRunOptions, stopCh <-chan struct{}) (*aggregatorapiserver.APIAggregator, error) {
	kubeAPIServerConfig, serviceResolver, pluginInitializer, err := CreateKubeAPIServerConfig(completedOptions)
	if err != nil {
		return nil, err
	}

	// 配置APIExtensionsServer， 并创建服务
	apiExtensionsConfig, err := createAPIExtensionsConfig(*kubeAPIServerConfig.GenericConfig, kubeAPIServerConfig.ExtraConfig.VersionedInformers, pluginInitializer, completedOptions.ServerRunOptions, completedOptions.MasterCount,
		serviceResolver, webhook.NewDefaultAuthenticationInfoResolverWrapper(kubeAPIServerConfig.ExtraConfig.ProxyTransport, kubeAPIServerConfig.GenericConfig.EgressSelector, kubeAPIServerConfig.GenericConfig.LoopbackClientConfig, kubeAPIServerConfig.GenericConfig.TracerProvider))
	if err != nil {
		return nil, err
	}
	notFoundHandler := notfoundhandler.New(kubeAPIServerConfig.GenericConfig.Serializer, genericapifilters.NoMuxAndDiscoveryIncompleteKey)
	apiExtensionsServer, err := createAPIExtensionsServer(apiExtensionsConfig, genericapiserver.NewEmptyDelegateWithCustomHandler(notFoundHandler))
	if err != nil {
		return nil, err
	}

	// 创建kubeAPIServerConfig服务
	kubeAPIServer, err := CreateKubeAPIServer(kubeAPIServerConfig, apiExtensionsServer.GenericAPIServer)
	if err != nil {
		return nil, err
	}

	// aggregator comes last in the chain
	// 配置 AggregatorServer，
	aggregatorConfig, err := createAggregatorConfig(*kubeAPIServerConfig.GenericConfig, completedOptions.ServerRunOptions, kubeAPIServerConfig.ExtraConfig.VersionedInformers, serviceResolver, kubeAPIServerConfig.ExtraConfig.ProxyTransport, pluginInitializer)
	if err != nil {
		return nil, err
	}
	aggregatorServer, err := createAggregatorServer(aggregatorConfig, kubeAPIServer.GenericAPIServer, apiExtensionsServer.Informers)
	if err != nil {
		// we don't need special handling for innerStopCh because the aggregator server doesn't create any go routines
		return nil, err
	}

	return aggregatorServer, nil
}
```

 `CreateKubeAPIServerConfig` 方法 -> `buildGenericConfig` -> `genericapiserver.NewConfig` -> `DefaultBuildHandlerChain`，在 `DefaultBuildHandlerChain` 方法中会对apiserver接口的链式判断，即俗称的filter操作。它的重点是每次链式判断会接受一个handler作为参数，并返回一个handler，作为下一次链式判断的参数。



创建KubeAPIServer时 ，在`Complete`方法中配置了各种参数，在`New`方法中根据提供的参数创建一个新的Master实例

```go
func CreateKubeAPIServer(kubeAPIServerConfig *controlplane.Config, delegateAPIServer genericapiserver.DelegationTarget) (*controlplane.Instance, error) {
	kubeAPIServer, err := kubeAPIServerConfig.Complete().New(delegateAPIServer)
	if err != nil {
		return nil, err
	}

	return kubeAPIServer, nil
}
```

正式运行

```go
d
```

#### 生命周期信号

kubernetes 生命周期信号总共有 6 个，其都是`lifecycleSignal`类型。

- `ShutdownInitiated` 当 apiserver 开始关机时，将会发出该信号。当主协程 的“stopCh” 收到终止信号并因此关闭时，它将会发出信号。
- `AfterShutdownDelayDuration` 在 ShutdownInitiated 后经过 ShutdownDelayDuration 长的时间后立即发出信号
- `InFlightRequestsDrained  ` 
- `HTTPServerStoppedListening` 
- `HasBeenReady` 
- `MuxAndDiscoveryComplete` 当所有已知的HTTP路径都已安装时，会发出MuxAndDiscoveryComplete信号。

在`namedChannelWrapper`中定义了一个通道，并实现 `lifecycleSignal` 包含的两个方法

-  `Signal`  关闭通道
- `Signaled` 一直返回一个相同的通道。当 `Signal` 被调用后，该通道会被关闭。





















### RESTful框架 go-restful

一个go-restful项目可以由多个container组成，每个container相当于一个单独的HTTP Server。一个container由多个We

bService组成。一个WebService由多个Handler组成。



`kubernetes/staging/src/k8s.io/apiserver/pkg/endpoints/discovery` 目录下的 `group.go` 、 `legacy.go` 、 `root.go`  `version.go` 实现了客户端通过Get请求获取相关资源信息。







`PathRecorderMux`













```go
func NewAPIServerHandler(name string, s runtime.NegotiatedSerializer, handlerChainBuilder HandlerChainBuilderFn, notFoundHandler http.Handler) *APIServerHandler {
	nonGoRestfulMux := mux.NewPathRecorderMux(name)
	if notFoundHandler != nil {
		nonGoRestfulMux.NotFoundHandler(notFoundHandler)
	}

	gorestfulContainer := restful.NewContainer()
	gorestfulContainer.ServeMux = http.NewServeMux()
	gorestfulContainer.Router(restful.CurlyRouter{}) // e.g. for proxy/{kind}/{name}/{*}
	gorestfulContainer.RecoverHandler(func(panicReason interface{}, httpWriter http.ResponseWriter) {
		logStackOnRecover(s, panicReason, httpWriter)
	})
	gorestfulContainer.ServiceErrorHandler(func(serviceErr restful.ServiceError, request *restful.Request, response *restful.Response) {
		serviceErrorHandler(s, serviceErr, request, response)
	})

	director := director{
		name:               name,
		goRestfulContainer: gorestfulContainer,
		nonGoRestfulMux:    nonGoRestfulMux,
	}

	return &APIServerHandler{
		FullHandlerChain:   handlerChainBuilder(director),
		GoRestfulContainer: gorestfulContainer,
		NonGoRestfulMux:    nonGoRestfulMux,
		Director:           director,
	}
}
```



参考list

- 《kubernetes源码剖析》
- kubernetes源码 1.23
