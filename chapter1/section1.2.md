# 启动流程

入口path: containerd/cmd/containerd/command/main.go

    app.Action = func(context *cli.Context) error {
    		var (
    			start   = time.Now()
    			signals = make(chan os.Signal, 2048)
    			serverC = make(chan *server.Server, 1)
    			ctx     = gocontext.Background()
    			config  = defaultConfig()
    			默认的 config root 为 /var/lib/containerd
    		)
    
    		done := handleSignals(ctx, signals, serverC)
    		// start the signal handler as soon as we can to make sure that
    		// we don't miss any signals during boot
    		signal.Notify(signals, handledSignals...)
    
    		if err := srvconfig.LoadConfig(context.GlobalString("config"), config); err != nil && !os.IsNotExist(err) {
    			return err
    		}
    		// apply flags to the config
    		if err := applyFlags(context, config); err != nil {
    			return err
    		}
    		// cleanup temp mounts
    		if err := mount.SetTempMountLocation(filepath.Join(config.Root, "tmpmounts")); err != nil {
    			return errors.Wrap(err, "creating temp mount location")
    		}
    		// unmount all temp mounts on boot for the server
    		warnings, err := mount.CleanupTempMounts(0)
    		if err != nil {
    			log.G(ctx).WithError(err).Error("unmounting temp mounts")
    		}
    		for _, w := range warnings {
    			log.G(ctx).WithError(w).Warn("cleanup temp mount")
    		}
    		address := config.GRPC.Address
    		if address == "" {
    			return errors.New("grpc address cannot be empty")
    		}
    		log.G(ctx).WithFields(logrus.Fields{
    			"version":  version.Version,
    			"revision": version.Revision,
    		}).Info("starting containerd")
    
    		server, err := server.New(ctx, config)
    		if err != nil {
    			return err
    		}
    		serverC <- server
    		if config.Debug.Address != "" {
    			var l net.Listener
    			if filepath.IsAbs(config.Debug.Address) {
    				if l, err = sys.GetLocalListener(config.Debug.Address, config.Debug.UID, config.Debug.GID); err != nil {
    					return errors.Wrapf(err, "failed to get listener for debug endpoint")
    				}
    			} else {
    				if l, err = net.Listen("tcp", config.Debug.Address); err != nil {
    					return errors.Wrapf(err, "failed to get listener for debug endpoint")
    				}
    			}
    			//初始化debug的接口
    			serve(ctx, l, server.ServeDebug)
    		}
    		if config.Metrics.Address != "" {
    			l, err := net.Listen("tcp", config.Metrics.Address)
    			if err != nil {
    				return errors.Wrapf(err, "failed to get listener for metrics endpoint")
    			}
    			//初始化ServeMetrics的接口
    			serve(ctx, l, server.ServeMetrics)
    		}
            
            //产生unix-socket被grpcServer使用来监听, 默认在/run/containerd/containerd.sock
    		l, err := sys.GetLocalListener(address, config.GRPC.UID, config.GRPC.GID)
    		if err != nil {
    			return errors.Wrapf(err, "failed to get listener for main endpoint")
    		}
    		//rpc在Server.New中被初始化
    		serve(ctx, l, server.ServeGRPC)
    
    		log.G(ctx).Infof("containerd successfully booted in %fs", time.Since(start).Seconds())
    		<-done
    		return nil
    	}
    	return app
    }



    // New creates and initializes a new containerd server
    func New(ctx context.Context, config *srvconfig.Config) (*Server, error) {
    	switch {
    	case config.Root == "":
    		return nil, errors.New("root must be specified")
    	case config.State == "":
    		return nil, errors.New("state must be specified")
    	case config.Root == config.State:
    		return nil, errors.New("root and state must be different paths")
    	}
    
    	if err := os.MkdirAll(config.Root, 0711); err != nil {
    		return nil, err
    	}
    	if err := os.MkdirAll(config.State, 0711); err != nil {
    		return nil, err
    	}
    	if err := apply(ctx, config); err != nil {
    		return nil, err
    	}
    	//加载插件，比如snapshottor, metaStore,content等，但是其中还有个proxyPlugin，不知道干啥的，有待补充？？？
    	plugins, err := LoadPlugins(ctx, config)
    	if err != nil {
    		return nil, err
    	}
    
    	serverOpts := []grpc.ServerOption{
    		grpc.UnaryInterceptor(grpc_prometheus.UnaryServerInterceptor),
    		grpc.StreamInterceptor(grpc_prometheus.StreamServerInterceptor),
    	}
    	if config.GRPC.MaxRecvMsgSize > 0 {
    		serverOpts = append(serverOpts, grpc.MaxRecvMsgSize(config.GRPC.MaxRecvMsgSize))
    	}
    	if config.GRPC.MaxSendMsgSize > 0 {
    		serverOpts = append(serverOpts, grpc.MaxSendMsgSize(config.GRPC.MaxSendMsgSize))
    	}
    	rpc := grpc.NewServer(serverOpts...)
    	var (
    		services []plugin.Service
    		s        = &Server{
    			rpc:    rpc,
    			events: exchange.NewExchange(),
    			config: config,
    		}
    		initialized = plugin.NewPluginSet()
    	)
    	//上边加载好了plugin后，由下边调用init方法初始化每个plugin，并将需要注册的grpc路由收集到services []plugin.Service中
    	for _, p := range plugins {
    		id := p.URI()
    		log.G(ctx).WithField("type", p.Type).Infof("loading plugin %q...", id)
    
    		initContext := plugin.NewContext(
    			ctx,
    			p,
    			initialized,
    			config.Root,
    			config.State,
    		)
    		initContext.Events = s.events
    		initContext.Address = config.GRPC.Address
    
    		// load the plugin specific configuration if it is provided
    		if p.Config != nil {
    			pluginConfig, err := config.Decode(p.ID, p.Config)
    			if err != nil {
    				return nil, err
    			}
    			initContext.Config = pluginConfig
    		}
    		result := p.Init(initContext)
    		if err := initialized.Add(result); err != nil {
    			return nil, errors.Wrapf(err, "could not add plugin result to plugin set")
    		}
    
    		instance, err := result.Instance()
    		if err != nil {
    			if plugin.IsSkipPlugin(err) {
    				log.G(ctx).WithField("type", p.Type).Infof("skip loading plugin %q...", id)
    			} else {
    				log.G(ctx).WithError(err).Warnf("failed to load plugin %s", id)
    			}
    			continue
    		}
    		// check for grpc services that should be registered with the server
    		if service, ok := instance.(plugin.Service); ok {
    			services = append(services, service)
    		}
    		s.plugins = append(s.plugins, result)
    	}
    	// register services after all plugins have been initialized(注册rpc路由信息)
    	for _, service := range services {
    		if err := service.Register(rpc); err != nil {
    			return nil, err
    		}
    	}
    	return s, nil
    }

下面简介一下plugin的类型

    const (
    	// InternalPlugin implements an internal plugin to containerd
    	InternalPlugin Type = "io.containerd.internal.v1"
    	// RuntimePlugin implements a runtime
    	RuntimePlugin Type = "io.containerd.runtime.v1"
    	// RuntimePluginV2 implements a runtime v2
    	RuntimePluginV2 Type = "io.containerd.runtime.v2"
    	// ServicePlugin implements a internal service
    	ServicePlugin Type = "io.containerd.service.v1"
    	// GRPCPlugin implements a grpc service
    	GRPCPlugin Type = "io.containerd.grpc.v1"
    	// SnapshotPlugin implements a snapshotter
    	SnapshotPlugin Type = "io.containerd.snapshotter.v1"
    	// TaskMonitorPlugin implements a task monitor
    	TaskMonitorPlugin Type = "io.containerd.monitor.v1"
    	// DiffPlugin implements a differ
    	DiffPlugin Type = "io.containerd.differ.v1"
    	// MetadataPlugin implements a metadata store
    	MetadataPlugin Type = "io.containerd.metadata.v1"
    	// ContentPlugin implements a content store
    	ContentPlugin Type = "io.containerd.content.v1"
    	// GCPlugin implements garbage collection policy
    	GCPlugin Type = "io.containerd.gc.v1"
    )

    func init() {
           plugin.Register(&plugin.Registration{
                  ID:   "btrfs",
                  Type: plugin.SnapshotPlugin,
                  Init: func(ic *plugin.InitContext) (interface{}, error) {
                         return NewSnapshotter(ic.Root)
                  },
           })
    }
    
    func init() {
           plugin.Register(&plugin.Registration{
                  Type: plugin.SnapshotPlugin,
                  ID:   "overlayfs",
                  Init: func(ic *plugin.InitContext) (interface{}, error) {
                         return NewSnapshotter(ic.Root)
                  },
           })
    }
    //可以看到containerd的snapshottor机制也是通过底层的这些storage driver实现的。
    
更详细的plugin介绍在后边涉及到的时候再深入研究。
