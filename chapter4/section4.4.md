# pouch commit实现

我们借助pouch commit的实现来看一下，如何使用containerd的diff和snapshot服务。
首先要明白containerd是不支持build/commit功能的，那么pouch是如何实现的commit命令呢？

### docker commit
要明白pouch commit如何实现,我们首先看一下docker commit是如何实现的，在功能上基本相关，只不过前者依赖containerd提供的grpc服务。

> 深入学习docker commit 的原理前，我不妨先来看看一下 docker help  中关于 commit 命令的阐述： commit
> Create a new image from a container's changes  结合上图与命令docker commit
> 的描述，我们可以发现有三个关键字Image、Container 与Changes 。如何理解这三个关键字，我们可以从以下三个步骤入手：

> 1. Docker Daemon 会通过一个 Docker 镜像运行一个 Docker 容器，Docker 通过层级文件系统为 Docker 容器提供文件系统视角，最上层的是可读可写层（Read-Write Layer）。
> 
> 2. Docker 容器初始的可读可写层内容均为空，Docker 容器对文件系统的内容更新将全部更新于可读可写层（Read-Write Layer）。
> 
> 3. 实现 docker commit 操作时，Docker 仅仅是将可读可写层（Read-Write Layer）中的更新内容，打包为一个全新的镜像。

简言之，所谓的commit功能就是将容器最上层的R/W层重新打包成一个新的镜像，至于说如何构造镜像要依赖不同的实现。


### pouch commit实现
有了上边的介绍，我们对commit的功能有了一个基本的了解，如果让我们在pouch里边去实现commit的话，无非依赖于以下几个操作，Diff操作来提取出R/W读写层，Content服务写入tar包，然后unpack到snapshot中去，我们实际看一下pouch commit的实现看是不是这样做的。

path:  alibaba/pouch/ctrd/image_commit.go

    func (c *Client) Commit(ctx context.Context, config *CommitConfig) (_ digest.Digest, err0 error) {
    	// get a containerd client
    	wrapperCli, err := c.Get(ctx)
    	if err != nil {
    		return "", fmt.Errorf("failed to get a containerd grpc client: %v", err)
    	}
    	client := wrapperCli.client
    
    	var (
    		sn     = client.SnapshotService(defaultSnapshotterName)
    		cs     = client.ContentStore()
    		differ = client.DiffService()
    	)
    
    	// export new layer
    	snapshot, err := c.GetSnapshot(ctx, config.ContainerID)
    	if err != nil {
    		return "", fmt.Errorf("failed to get snapshot: %s", err)
    	}
    	// 调用diff服务，返回描述这一层的descriptor
    	layer, diffIDStr, err := exportLayer(ctx, snapshot.Name, sn, cs, differ)
    	if err != nil {
    		return "", err
    	}
    
    	// create child image
    	diffIDDigest, err := digest.Parse(diffIDStr)
    	if err != nil {
    		return "", err
    	}
        
        //结合parent image构建一个符合oci spec的img描述体
    	childImg, err := newChildImage(ctx, config, diffIDDigest)
    	if err != nil {
    		return "", err
    	}
    
    	// create new snapshot for new layer
    	//产生新镜像chainId作为镜像的snapshotKey
    	snapshotKey := identity.ChainID(childImg.RootFS.DiffIDs).String()
    	//还是按照pull镜像时候的3步走 先prepare -> apply -> commit
    	if err = newSnapshot(ctx, config.Image, sn, differ, layer, snapshotKey, diffIDStr); err != nil {
    		return "", err
    	}
    	defer func() {
    		if err0 != nil {
    			logrus.Warnf("remove snapshot %s cause commit image failed", snapshotKey)
    			client.SnapshotService(defaultSnapshotterName).Remove(ctx, snapshotKey)
    		}
    	}()
    
    	imgJSON, err := json.Marshal(childImg)
    	if err != nil {
    		return "", err
    	}
    	
    	// 以下分别构建新镜像config descriptor的layer descriptor
    
    	// new config descriptor
    	configDesc := ocispec.Descriptor{
    		MediaType: configType,
    		Digest:    digest.FromBytes(imgJSON),
    		Size:      int64(len(imgJSON)),
    	}
    
    	// get parent image layer descriptor
    	pmfst, err := images.Manifest(ctx, cs, config.CImage.Target(), platforms.Default())
    	if err != nil {
    		return "", err
    	}
    
    	// new layer descriptor
    	layers := append(pmfst.Layers, layer)
    	labels := map[string]string{
    		"containerd.io/gc.ref.content.0": configDesc.Digest.String(),
    	}
    	for i, l := range layers {
    		labels[fmt.Sprintf("containerd.io/gc.ref.content.%d", i+1)] = l.Digest.String()
    	}
    
    	// new manifest descriptor
    	mfst := ocispec.Manifest{
    		Versioned: specs.Versioned{
    			SchemaVersion: 2,
    		},
    		Config: configDesc,
    		Layers: layers,
    	}
    
    	mfstJSON, err := json.MarshalIndent(mfst, "", "   ")
    	if err != nil {
    		return "", errors.Wrap(err, "failed to marshal manifest")
    	}
    
    	mfstDigest := digest.FromBytes(mfstJSON)
    	mfstDesc := ocispec.Descriptor{
    		Digest: mfstDigest,
    		Size:   int64(len(mfstJSON)),
    	}
    
    	desc := ocispec.Descriptor{
    		MediaType: manifestType,
    		Digest:    mfstDigest,
    		Size:      int64(len(mfstJSON)),
    	}
    
    	// image create
    	img := images.Image{
    		Name:      config.Reference,
    		Target:    desc,
    		CreatedAt: time.Now(),
    	}
    
    	// register containerd image metadata.
    	// 向containerd的images注册镜像的元数据
    	if _, err := client.ImageService().Update(ctx, img); err != nil {
    		if !errdefs.IsNotFound(err) {
    			return "", fmt.Errorf("failed to cover exist image %s", err)
    		}
    		if _, err := client.ImageService().Create(ctx, img); err != nil {
    			return "", fmt.Errorf("failed to create new image %s", err)
    		}
    	}
    	
    
    	// write manifest content
    	//镜像为单位的，在unpack那个环节有介绍
    	if err := content.WriteBlob(ctx, cs, mfstDigest.String(), bytes.NewReader(mfstJSON), mfstDesc.Size, mfstDesc.Digest, content.WithLabels(labels)); err != nil {
    		return "", errors.Wrapf(err, "error writing manifest blob %s", mfstDigest)
    	}
    
    	// write config content
    	// 实际上就是前边那个oci spec img
    	if err := content.WriteBlob(ctx, cs, configDesc.Digest.String(), bytes.NewReader(imgJSON), configDesc.Size, configDesc.Digest); err != nil {
    		return "", errors.Wrap(err, "error writing config blob")
    	}
    
    	// pouch record config descriptor digest as image id.
    	return configDesc.Digest, nil
    }

### 总结
通过观察commit的实现来说，我们可以发现diff服务和snapshot服务之间的职责分工，diff服务会操作tar包来完成r/w的打包，applyDiff这些操作，而snapshot服务来说完成类似登记的功能，以mounts挂载点来和其他服务连接起来。

### 参考
https://github.com/alibaba/pouch/pull/2125
http://guide.daocloud.io/dcs/docker-commit-9153991.html
