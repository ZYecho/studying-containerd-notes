# Containers API


入口path： containerd/api/services，其中的每个service中都定义各自负责模块的rpc接口。

### List
以containerd list为例子
首先是函数入口:

    func _Containers_List_Handler(srv interface{}, ctx context.Context, dec func(interface{}) error, interceptor grpc.UnaryServerInterceptor) (interface{}, error) {
           in := new(ListContainersRequest)
           if err := dec(in); err != nil {
                  return nil, err
           }
           if interceptor == nil {
                  return srv.(ContainersServer).List(ctx, in)
           }
           info := &grpc.UnaryServerInfo{
                  Server:     srv,
                  FullMethod: "/containerd.services.containers.v1.Containers/List",
           }
           //这个handler就是API具体的处理逻辑
           handler := func(ctx context.Context, req interface{}) (interface{}, error) {
                  return srv.(ContainersServer).List(ctx, req.(*ListContainersRequest))
           }
           return interceptor(ctx, in, info, handler)
    }

对应接口的实现都在path: containerd/services，对应service的plugin的path在: containerd/services/containers/service.go, 通过对应Service的client为上层提供调用接口，上文中提到的List接口，在path: containerd/services/containers/local.go中，

    func (l *local) List(ctx context.Context, req *api.ListContainersRequest, _ ...grpc.CallOption) (*api.ListContainersResponse, error) {
    	var resp api.ListContainersResponse
    	return &resp, errdefs.ToGRPC(l.withStoreView(ctx, func(ctx context.Context, store containers.Store) error {
    		containers, err := store.List(ctx, req.Filters...)
    		if err != nil {
    			return err
    		}
    		resp.Containers = containersToProto(containers)
    		return nil
    	}))
    }
    
根据 withStore 函数可以得到 store 为 metadata.NewContainerStore，路径 containerd/metadata/containers.go 中，containerStore 结构体是包裹的是操作数据库。

    func (s *containerStore) List(ctx context.Context, fs ...string) ([]containers.Container, error) {
    	namespace, err := namespaces.NamespaceRequired(ctx)
    	if err != nil {
    		return nil, err
    	}
    
    	filter, err := filters.ParseAll(fs...)
    	if err != nil {
    		return nil, errors.Wrapf(errdefs.ErrInvalidArgument, err.Error())
    	}
    
    	bkt := getContainersBucket(s.tx, namespace)
    	if bkt == nil {
    		return nil, nil // empty store
    	}
    
    	var m []containers.Container
    	if err := bkt.ForEach(func(k, v []byte) error {
    		cbkt := bkt.Bucket(k)
    		if cbkt == nil {
    			return nil
    		}
    		container := containers.Container{ID: string(k)}
            //从boltdb的存储格式转化为Container结构
    		if err := readContainer(&container, cbkt); err != nil {
    			return errors.Wrapf(err, "failed to read container %q", string(k))
    		}
    
    		if filter.Match(adaptContainer(container)) {
    			m = append(m, container)
    		}
    		return nil
    	}); err != nil {
    		return nil, err
    	}
    
    	return m, nil
    }


    func getContainersBucket(tx *bolt.Tx, namespace string) *bolt.Bucket {
    	return getBucket(tx, bucketKeyVersion, []byte(namespace), bucketKeyObjectContainers)
    }
    
    //是个嵌套的bucket结构，从最后的name为:bucketKeyObjectContainers的Bucket中取出value
    func getBucket(tx *bolt.Tx, keys ...[]byte) *bolt.Bucket {
	    bkt := tx.Bucket(keys[0])

	    for _, key := range keys[1:] {
		    if bkt == nil {
			    break
		    }
		    bkt = bkt.Bucket(key)
	    }
	    return bkt
	}
    
### Create
入口

    func _Containers_Create_Handler(srv interface{}, ctx context.Context, dec func(interface{}) error, interceptor grpc.UnaryServerInterceptor) (interface{}, error) {
    	in := new(CreateContainerRequest)
    	if err := dec(in); err != nil {
    		return nil, err
    	}
    	if interceptor == nil {
    		return srv.(ContainersServer).Create(ctx, in)
    	}
    	info := &grpc.UnaryServerInfo{
    		Server:     srv,
    		FullMethod: "/containerd.services.containers.v1.Containers/Create",
    	}
    	handler := func(ctx context.Context, req interface{}) (interface{}, error) {
    		return srv.(ContainersServer).Create(ctx, req.(*CreateContainerRequest))
    	}
    	return interceptor(ctx, in, info, handler)
    }
    
执行逻辑同样位于在path: containerd/services/containers/local.go中

    func (l *local) Create(ctx context.Context, req *api.CreateContainerRequest, _ ...grpc.CallOption) (*api.CreateContainerResponse, error) {
    	var resp api.CreateContainerResponse
    
    	if err := l.withStoreUpdate(ctx, func(ctx context.Context, store containers.Store) error {
    		container := containerFromProto(&req.Container)
    
    		created, err := store.Create(ctx, container)
    		if err != nil {
    			return err
    		}
    
    		resp.Container = containerToProto(&created)
    
    		return nil
    	}); err != nil {
    		return &resp, errdefs.ToGRPC(err)
    	}
    	// 发出事件Event
    	if err := l.publisher.Publish(ctx, "/containers/create", &eventstypes.ContainerCreate{
    		ID:    resp.Container.ID,
    		Image: resp.Container.Image,
    		Runtime: &eventstypes.ContainerCreate_Runtime{
    			Name:    resp.Container.Runtime.Name,
    			Options: resp.Container.Runtime.Options,
    		},
    	}); err != nil {
    		return &resp, err
    	}
    
    	return &resp, nil
    }

//最后落到存储层

    func (s *containerStore) Create(ctx context.Context, container containers.Container) (containers.Container, error) {
    	namespace, err := namespaces.NamespaceRequired(ctx)
    	if err != nil {
    		return containers.Container{}, err
    	}
    
    	if err := validateContainer(&container); err != nil {
    		return containers.Container{}, errors.Wrap(err, "create container failed validation")
    	}
        
        //如果没有创建过才去创建
    	bkt, err := createContainersBucket(s.tx, namespace)
    	if err != nil {
    		return containers.Container{}, err
    	}
    
        //又去创建了个子bucket，在containersBucket下边
    	cbkt, err := bkt.CreateBucket([]byte(container.ID))
    	if err != nil {
    		if err == bolt.ErrBucketExists {
    			err = errors.Wrapf(errdefs.ErrAlreadyExists, "container %q", container.ID)
    		}
    		return containers.Container{}, err
    	}
    
    	container.CreatedAt = time.Now().UTC()
    	container.UpdatedAt = container.CreatedAt
    	if err := writeContainer(cbkt, &container); err != nil {
    		return containers.Container{}, errors.Wrapf(err, "failed to write container %q", container.ID)
    	}
    
    	return container, nil
    }

综上所述，Containers相关的接口基本上是在维护容器相关的metadata.想要创建完整的容器还需要依赖其他的接口比如Task相关的接口。

> A container is a metadata object that resources are allocated and
> attached to




