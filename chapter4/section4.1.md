# image fetch

如下图所展示pull images的流程，总体来说containerd的pull流程可以分为fetch和unpack两部分。本节主要关注fetch步骤。

![image.png-351.2kB][1]

### oci distribution spec
目前containerd当前同时支持docker版的和oci版的registry api。我们首先来看一下oci distribution spec定义了哪些内容。

path: https://github.com/opencontainers/distribution-spec/blob/master/spec.md

    An "image" is a combination of a JSON manifest and individual layer files. The process of pulling an image centers around retrieving these two components.

想要pull image首先要从registry中获取manifests，包含以下的fields.

| field     | description                                    |
|-----------|------------------------------------------------|
| name      | The name of the image.                         |
| tag       | The tag for this version of the image.         |
| fsLayers  | A list of layer descriptors (including digest) |
| signature | A JWS used to verify the manifest content      |

更详细的manifest的example可以在`https://github.com/ZYecho/image-spec/blob/master/manifest.md`中找到。
对应的API为`GET /v2/<name>/manifests/<reference>`，the reference may include a tag or digest.

然后根据manifests中包含的digest去获取每层layer的blob，通过API `GET /v2/<name>/blobs/<digest>`进行获取。

值得注意在整个拉取过程中没有出现image id的用处，image id是docker的用法，image id和manifest中config的digest相同，本质上是image configuration JSON的digest。

Docker-Content-Digest header：
这个头部后边还需要用到，先放到这里了解一下。
> To provide verification of http content, any response may include a
> Docker-Content-Digest header. This will include the digest of the
> target entity returned in the response. For blobs, this is the entire
> blob content. For manifests, this is the manifest body without the
> signature content, also known as the JWS payload. Note that the
> commonly used canonicalization for digest calculation may be dependent
> on the mediatype of the content, such as with manifests.

### fetch流程
path: containerd/cmd/ctr/commands/images/pull.go
ctr image pull操作流程如下：

 1. resolve用户需要下载的镜像 
 2. 从registry pull镜像，把镜像层内容和config保存进content服务，把镜像相关的元数据保存进images元数据服务
 3. unpack进snapshot服务
    
    //ref就是拉取用的reference, 形如docker.io/library/redis:alpine
    img, err := content.Fetch(ctx, client, ref, config)
    if err != nil {
    	return err
    }
    
这里直接调用content实现的Fetch函数，是和`ctr content fetch`用的同样的处理逻辑。 

    // Fetch loads all resources into the content store and returns the image
    func Fetch(ctx context.Context, client *containerd.Client, ref string, config *FetchConfig) (images.Image, error) {
    	ongoing := newJobs(ref)
    
    	pctx, stopProgress := context.WithCancel(ctx)
    	progress := make(chan struct{})
    
    	go func() {
    		if config.ProgressOutput != nil {
    			// no progress bar, because it hides some debug logs
    			showProgress(pctx, ongoing, client.ContentStore(), config.ProgressOutput)
    		}
    		close(progress)
    	}()
    
    	h := images.HandlerFunc(func(ctx context.Context, desc ocispec.Descriptor) ([]ocispec.Descriptor, error) {
    	    // 将非manifest的digest添加到Ongoing中，在showProgress中会使用到
    		if desc.MediaType != images.MediaTypeDockerSchema1Manifest {
    			ongoing.add(desc)
    		}
    		return nil, nil
    	})
    
    	log.G(pctx).WithField("image", ref).Debug("fetching")
    	labels := commands.LabelArgs(config.Labels)
    	opts := []containerd.RemoteOpt{
    		containerd.WithPullLabels(labels),
    		containerd.WithResolver(config.Resolver),
    		containerd.WithImageHandler(h),
    		containerd.WithSchema1Conversion,
    	}
    	for _, platform := range config.Platforms {
    		opts = append(opts, containerd.WithPlatform(platform))
    	}
    	//调用containerd的grpc服务
    	img, err := client.Fetch(pctx, ref, opts...)
    	stopProgress()
    	if err != nil {
    		return images.Image{}, err
    	}
    
    	<-progress
    	return img, nil
    }

path: containerd/containerd/client.go

    func (c *Client) fetch(ctx context.Context, rCtx *RemoteContext, ref string, limit int) (images.Image, error) {
    	store := c.ContentStore()
    	//第一步首先将ref解析为descriptor,descriptor可以理解为一个可以用来下载的描述对象
    	name, desc, err := rCtx.Resolver.Resolve(ctx, ref)
    	if err != nil {
    		return images.Image{}, errors.Wrapf(err, "failed to resolve reference %q", ref)
    	}
    
    	fetcher, err := rCtx.Resolver.Fetcher(ctx, name)
    	if err != nil {
    		return images.Image{}, errors.Wrapf(err, "failed to get fetcher for %q", name)
    	}
    
    	var (
    		schema1Converter *schema1.Converter
    		handler          images.Handler
    	)
    	if desc.MediaType == images.MediaTypeDockerSchema1Manifest && rCtx.ConvertSchema1 {
    	    // 兼容逻辑
    		schema1Converter = schema1.NewConverter(store, fetcher)
    		handler = images.Handlers(append(rCtx.BaseHandlers, schema1Converter)...)
    	} else {
    		// Get all the children for a descriptor
    		childrenHandler := images.ChildrenHandler(store)
    		// Set any children labels for that content
    		childrenHandler = images.SetChildrenLabels(store, childrenHandler)
    		// Filter children by platforms
    		childrenHandler = images.FilterPlatforms(childrenHandler, rCtx.PlatformMatcher)
    		// Sort and limit manifests if a finite number is needed
    		if limit > 0 {
    			childrenHandler = images.LimitManifests(childrenHandler, rCtx.PlatformMatcher, limit)
    		}
            // 会在后边分别介绍这几个Handler, Handlers返回一个新的Handler(具备顺序遍历的功能)
    		handler = images.Handlers(append(rCtx.BaseHandlers,
    			remotes.FetchHandler(store, fetcher),
    			childrenHandler,
    		)...)
    	}
        
        //这个真正开始下载的流程，包括manifest以及镜像层
    	if err := images.Dispatch(ctx, handler, desc); err != nil {
    		return images.Image{}, err
    	}
    	if schema1Converter != nil {
    		desc, err = schema1Converter.Convert(ctx)
    		if err != nil {
    			return images.Image{}, err
    		}
    	}
    
    	img := images.Image{
    		Name:   name,
    		Target: desc,
    		Labels: rCtx.Labels,
    	}
    
    	is := c.ImageService()
    	for {
    		if created, err := is.Create(ctx, img); err != nil {
    			if !errdefs.IsAlreadyExists(err) {
    				return images.Image{}, err
    			}
    
    			updated, err := is.Update(ctx, img)
    			if err != nil {
    				// if image was removed, try create again
    				if errdefs.IsNotFound(err) {
    					continue
    				}
    				return images.Image{}, err
    			}
    
    			img = updated
    		} else {
    			img = created
    		}
    
    		return img, nil
    	}
    }

### 构造descriptor对象
下边看一下第一步ref是如何转到为descriptor的。

    func (r *dockerResolver) Resolve(ctx context.Context, ref string) (string, ocispec.Descriptor, error) {
       //首先将ref转化为locator(docker.io/library/redis)和object(tag -> 2.7.8或者是digest)两部分
    	refspec, err := reference.Parse(ref)
    	if err != nil {
    		return "", ocispec.Descriptor{}, err
    	}
    
    	if refspec.Object == "" {
    		return "", ocispec.Descriptor{}, reference.ErrObjectRequired
    	}
        
        //在其中提取出了镜像repo的地址，为了后期方便组装镜像的地址
    	base, err := r.base(refspec)
    	if err != nil {
    		return "", ocispec.Descriptor{}, err
    	}
    
    	fetcher := dockerFetcher{
    		dockerBase: base,
    	}
    
    	var (
    		urls []string
    		dgst = refspec.Digest()
    	)
        
        //如果是Object是用digest来表示的
    	if dgst != "" {
    		if err := dgst.Validate(); err != nil {
    			// need to fail here, since we can't actually resolve the invalid
    			// digest.
    			return "", ocispec.Descriptor{}, err
    		}
            
            //首先尝试registry-1.docker.io/v2/library/ubuntu/manifest/sha256:xxx
    		// turns out, we have a valid digest, make a url.
    		urls = append(urls, fetcher.url("manifests", dgst.String()))
    
    		// fallback to blobs on not found.
    		//如果失败再使用registry-1.docker.io/v2/library/ubuntu/blobs/sha256:xxx
    		urls = append(urls, fetcher.url("blobs", dgst.String()))
    	} else {
    	    //直接使用tag来进行访问，registry-1.docker.io/v2/library/redis/manifest/alpine
    		urls = append(urls, fetcher.url("manifests", refspec.Object))
    	}
    
    	ctx, err = contextWithRepositoryScope(ctx, refspec, false)
    	if err != nil {
    		return "", ocispec.Descriptor{}, err
    	}
    	for _, u := range urls {
    	    //注意这里是HEAD，也就是对应着spec中的Existing Manifests API，先看看Manifests在不在，在的话再去下载之
    		req, err := http.NewRequest(http.MethodHead, u, nil)
    		if err != nil {
    			return "", ocispec.Descriptor{}, err
    		}
    
    		// set headers for all the types we support for resolution.
    		//构建HTTP请求对象头部
    		req.Header.Set("Accept", strings.Join([]string{
    			images.MediaTypeDockerSchema2Manifest,
    			images.MediaTypeDockerSchema2ManifestList,
    			ocispec.MediaTypeImageManifest,
    			ocispec.MediaTypeImageIndex, "*"}, ", "))
    
    		log.G(ctx).Debug("resolving")
    		resp, err := fetcher.doRequestWithRetries(ctx, req, nil)
    		if err != nil {
    			if errors.Cause(err) == ErrInvalidAuthorization {
    				err = errors.Wrapf(err, "pull access denied, repository does not exist or may require authorization")
    			}
    			return "", ocispec.Descriptor{}, err
    		}
    		//在构建descriptor过程中没有使用到resp body的内容
    		resp.Body.Close() // don't care about body contents.
    
    		if resp.StatusCode > 299 {
    			if resp.StatusCode == http.StatusNotFound {
    				continue
    			}
    			return "", ocispec.Descriptor{}, errors.Errorf("unexpected status code %v: %v", u, resp.Status)
    		}
    
    		// this is the only point at which we trust the registry. we use the
    		// content headers to assemble a descriptor for the name. when this becomes
    		// more robust, we mostly get this information from a secure trust store.
    		//关于这个头部，在前边介绍OCI distribution spec的时候有提到，目的还是为了校验
    		dgstHeader := digest.Digest(resp.Header.Get("Docker-Content-Digest"))
    
    		if dgstHeader != "" {
    			if err := dgstHeader.Validate(); err != nil {
    				return "", ocispec.Descriptor{}, errors.Wrapf(err, "%q in header not a valid digest", dgstHeader)
    			}
    			dgst = dgstHeader
    		}
    
    		if dgst == "" {
    			return "", ocispec.Descriptor{}, errors.Errorf("could not resolve digest for %v", ref)
    		}
    
    		var (
    			size       int64
    			sizeHeader = resp.Header.Get("Content-Length")
    		)
    
    		size, err = strconv.ParseInt(sizeHeader, 10, 64)
    		if err != nil {
    
    			return "", ocispec.Descriptor{}, errors.Wrapf(err, "invalid size header: %q", sizeHeader)
    		}
    		if size < 0 {
    			return "", ocispec.Descriptor{}, errors.Errorf("%q in header not a valid size", sizeHeader)
    		}
    
    		desc := ocispec.Descriptor{
    		    //可以看到这三个值都是从头部中拿到的
    			Digest:    dgst,
    			//可能为application/vnd.docker.distribution.manifest.list.v2+json, 对应着不同平台的多个image, containerd在向registry下载manifest list之后，再去选择下载其中的某个平台的镜像。
    			MediaType: resp.Header.Get("Content-Type"), // need to strip disposition?
    			Size:      size,
    		}
    
    		log.G(ctx).WithField("desc.digest", desc.Digest).Debug("resolved")
    		return ref, desc, nil
    	}
    
    	return "", ocispec.Descriptor{}, errors.Errorf("%v not found", ref)
    }

    //执行下载的主要框架，具体的执行逻辑在handler中，在下一节进行介绍
    func Dispatch(ctx context.Context, handler Handler, descs ...ocispec.Descriptor) error {
    	eg, ctx := errgroup.WithContext(ctx)
    	for _, desc := range descs {
    		desc := desc
    
    		eg.Go(func() error {
    			desc := desc
    
    			children, err := handler.Handle(ctx, desc)
    			if err != nil {
    				if errors.Cause(err) == ErrSkipDesc {
    					return nil // don't traverse the children.
    				}
    				return err
    			}
    
    			if len(children) > 0 {
    			    // 本质上是个dfs过程
    				return Dispatch(ctx, handler, children...)
    			}
    
    			return nil
    		})
    	}
    
    	return eg.Wait()
    }

#### Handlers介绍

下载的主要是任务都是通过Handler来完成的，看一下其最原始的定义。
path: /home/zhangyue/go/src/github.com/containerd/containerd/images/handlers.go, 也关注一下其中的一些utils函数

    // HandlerFunc function implementing the Handler interface
    type HandlerFunc func(ctx context.Context, desc ocispec.Descriptor) (subdescs []ocispec.Descriptor, err error)

BaseHandler:
通过connext注册过来的，只是负责将当前的desc进行登记，

    h := images.HandlerFunc(func(ctx context.Context, desc ocispec.Descriptor) ([]ocispec.Descriptor, error) {
    	if desc.MediaType != images.MediaTypeDockerSchema1Manifest {
    		ongoing.add(desc)
    	}
    	return nil, nil
    })

FetchHandler: containerd/containerd/remotes/handlers.go

        func fetch(ctx context.Context, ingester content.Ingester, fetcher Fetcher, desc ocispec.Descriptor) error {
    	log.G(ctx).Debug("fetch")
    
    	cw, err := content.OpenWriter(ctx, ingester, content.WithRef(MakeRefKey(ctx, desc)), content.WithDescriptor(desc))
    	if err != nil {
    		if errdefs.IsAlreadyExists(err) {
    			return nil
    		}
    		return err
    	}
    	defer cw.Close()
    
    	ws, err := cw.Status()
    	if err != nil {
    		return err
    	}
    
    	if ws.Offset == desc.Size {
    		// If writer is already complete, commit and return
    		err := cw.Commit(ctx, desc.Size, desc.Digest)
    		if err != nil && !errdefs.IsAlreadyExists(err) {
    			return errors.Wrapf(err, "failed commit on ref %q", ws.Ref)
    		}
    		return nil
    	}
        
        //通过http get得到Reader
    	rc, err := fetcher.Fetch(ctx, desc)
    	if err != nil {
    		return err
    	}
    	defer rc.Close()
        
        //将得到的rc写入到content中去
    	return content.Copy(ctx, cw, rc, desc.Size, desc.Digest)
    }

path: containerd/containerd/remotes/docker/fetcher.go

    func (r dockerFetcher) Fetch(ctx context.Context, desc ocispec.Descriptor) (io.ReadCloser, error) {
    	ctx = log.WithLogger(ctx, log.G(ctx).WithFields(
    		logrus.Fields{
    			"base":   r.base.String(),
    			"digest": desc.Digest,
    		},
    	))
       // 如果是manifest的话首先是*/manifests/*
       // 其次是走*/blobs/*即可
       // 如果不是manifest的话直接走*/blobs/*即可
    	urls, err := r.getV2URLPaths(ctx, desc)
    	if err != nil {
    		return nil, err
    	}
    
    	ctx, err = contextWithRepositoryScope(ctx, r.refspec, false)
    	if err != nil {
    		return nil, err
    	}
    
    	return newHTTPReadSeeker(desc.Size, func(offset int64) (io.ReadCloser, error) {
    		for _, u := range urls {
    			rc, err := r.open(ctx, u, desc.MediaType, offset)
    			if err != nil {
    				if errdefs.IsNotFound(err) {
    					continue // try one of the other urls.
    				}
    
    				return nil, err
    			}
    
    			return rc, nil
    		}
    
    		return nil, errors.Wrapf(errdefs.ErrNotFound,
    			"could not fetch content descriptor %v (%v) from remote",
    			desc.Digest, desc.MediaType)
    
    	})
    }

childrenHandler:

    // Get all the children for a descriptor
    // 这是个解析manifest的handler，返回[]Descriptor
    childrenHandler := images.ChildrenHandler(store)
    // Set any children labels for that content
    childrenHandler = images.SetChildrenLabels(store, childrenHandler)
    // Filter children by platforms
    childrenHandler = images.FilterPlatforms(childrenHandler, pullCtx.Platforms...)

中间的处理handler, path: containerd/containerd/images/handlers.go

    func SetChildrenLabels(manager content.Manager, f HandlerFunc) HandlerFunc {
        return func(ctx context.Context, desc ocispec.Descriptor) ([]ocispec.Descriptor, error) {
            children, err := f(ctx, desc)
            if err != nil {
                return children, err
            }
    
            if len(children) &gt; 0 {//如果在上一步处理结束后发现包含子descriptor
                info := content.Info{
                    Digest: desc.Digest,
                    Labels: map[string]string{},
                }
                fields := []string{}
                for i, ch := range children {
                    info.Labels[fmt.Sprintf("containerd.io/gc.ref.content.%d", i)] = ch.Digest.String()
                    fields = append(fields, fmt.Sprintf("labels.containerd.io/gc.ref.content.%d", i))
                }
                //将子descriptor作为descriptor的label
                _, err := manager.Update(ctx, info, fields...)
                if err != nil {
                    return nil, err
                }
            }
    
            return children, err
        }
    }

    //主要作用是在manifest list上添加label来表示包含关系
    func SetChildrenLabels(manager content.Manager, f HandlerFunc) HandlerFunc {
        return func(ctx context.Context, desc ocispec.Descriptor) ([]ocispec.Descriptor, error) {
            children, err := f(ctx, desc)
            if err != nil {
                return children, err
            }
     
            if len(children) &gt; 0 {//如果在上一步处理结束后发现包含子descriptor
                info := content.Info{
                    Digest: desc.Digest,
                    Labels: map[string]string{},
                }
                fields := []string{}
                for i, ch := range children {
                    info.Labels[fmt.Sprintf("containerd.io/gc.ref.content.%d", i)] = ch.Digest.String()
                    fields = append(fields, fmt.Sprintf("labels.containerd.io/gc.ref.content.%d", i))
                }
                //将子descriptor作为descriptor的label
                _, err := manager.Update(ctx, info, fields...)
                if err != nil {
                    return nil, err
                }
            }
     
            return children, err
        }
    }

要注意这些handler在使用过程中是按照顺序来调用，因此首先是baseHandler将任务添加到onGoning任务列表，然后是fetchHandler完成descreiptor的下载任务(落入content对象)，最后是childrenHandler的3个子handler完成任务(完成解析，打标签，过滤的操作); 同时要注意dispatch函数是一个深度优先遍历的过程，同时其内部如果有多个descriptor的话，那么本身是一个并行的过程，也就是说不同layer之间实际上是并行下载的。

### 参考
http://www.sel.zju.edu.cn/?p=921

  [1]: http://static.zybuluo.com/myecho/5ge7ybzdsb69920sxvkd958k/image.png
