# image unpack

![pull image][1]
前边介绍过fetch把镜像层内容和config保存进content服务，把镜像相关的元数据保存进images元数据服务中，而unpack过程中如下所述

> Once the image is pulled, the user can instruct the bundle controller
> to unpack the image into a bundle. Consuming from the content store,
> layers from the image are unpacked into the snapshot component.

### OCI image spec
![镜像之间关系图][2]
Image Index: 可以理解为Manifest list, 是镜像在不同平台的Manifest的集合， path: https://github.com/ZYecho/image-spec/blob/master/image-index.md

Manifest是一个镜像描述文件，包含config和layers两个主体，path:https://github.com/ZYecho/image-spec/blob/master/manifest.md

layers有很多层，其digest对应压缩后的格式比如gzip，tar的sha256值。

这里要区分下Manifest和镜像的描述文件的区别，下边给出两个例子

// Manifest文件，其digest在Image Index中

    {
    "schemaVersion": 2,
       "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
       "config": {
          "mediaType": "application/vnd.docker.container.image.v1+json",
          "size": 5108,
          //对应config文件的digest，需要再去下载
          "digest": "sha256:80581db8c700155a91bec6fd6398dad9733135e7c58a19472aa679e8367692ab"
       },
       "layers": [
          {
             "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
             "size": 2206542,
             "digest": "sha256:8e3ba11ec2a2b39ab372c60c16b421536e50e5ce64a0bc81765c2e38381bcff6"
          },
          {
             "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
             "size": 1249,
             "digest": "sha256:1f20bd2a5c234ffab42de6cbf83522946614b21b642a8208dca6b0fd614c31db"
          },
          {
             "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
             "size": 9071,
             "digest": "sha256:782ff7702b5cd0a7c0109740838c74945fc27e4ce34e1028c24bf73f8249a63a"
          },
          {
             "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
             "size": 9407568,
             "digest": "sha256:cd719ead7ee305492514a8dfa2afcd0979a16e8192836b4aaed98d8d932973c0"
          },
          {
             "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
             "size": 98,
             "digest": "sha256:01018940af9a67873ad6737c275cb134372cdf1cda565af58dd14a1e3b85ab2a"
          },
          {
             "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
             "size": 398,
             "digest": "sha256:3f1bfdda9588f5c0643b485580060b460b21b5331f4760778ef3279680e20966"
          }
       ]
    }

  这是config文件，path: https://github.com/ZYecho/image-spec/blob/master/config.md
  
    {
       "architecture":"amd64",
       "config":{
          "Hostname":"",
          "Domainname":"",
          "User":"",
          "AttachStdin":false,
          "AttachStdout":false,
          "AttachStderr":false,
          "ExposedPorts":{
             "6379/tcp":{
     
             }
          },
        ...
       "history":[
          {
             "created":"2018-07-06T14:14:06.165546783Z",
             "created_by":"/bin/sh -c #(nop) ADD file:25f61d70254b9807a40cd3e8d820f6a5ec0e1e596de04e325f6a33810393e95a in / "
          },
          {
             "created":"2018-07-11T00:55:43.769226605Z",
             "created_by":"/bin/sh -c apk add --no-cache 'su-exec\u003e=0.2'"
          },
          {
             "created":"2018-07-11T00:57:21.027940339Z",
             "created_by":"/bin/sh -c #(nop)  VOLUME [/data]",
             "empty_layer":true
          },
          {
             "created":"2018-07-11T00:57:21.312115762Z",
             "created_by":"/bin/sh -c #(nop) WORKDIR /data",
             "empty_layer":true
          }
       ],
       "os":"linux",
       //这个很重要
       "rootfs":{
          "type":"layers",
          "diff_ids":[
             "sha256:73046094a9b835e443af1a9d736fcfc11a994107500e474d0abf399499ed280c",
             "sha256:9f8f870604a08909589f09337944210db2bf72b2a71f0f707642b3aa9d225f9b",
             "sha256:221a0f51690d5f9063c9a113aa5e1340b50ac4474e7525efb9c60b945589110f",
             "sha256:1b44c45cbb1c6711042de1c2e8785b554da0d25ed03bbd2a6a13fb498eceb6ae",
             "sha256:211a9f3eb69c634f0368994600e7c93df7510f87870da57e51460f177c603ca9",
             "sha256:151bcb0152a97b3a445bc2e2ed29432fc77d662fcb746b05b87d77a8d7bbf023"
          ]
       }
    }

注意区别一下diffid和manifest layer digest的区别:

> A layer DiffID is the digest over the layer's uncompressed tar archive
> and serialized in the descriptor digest format
> 
> NOTE: Do not confuse DiffIDs with layer digests, often referenced in
> the manifest, which are digests over compressed or uncompressed
> content.


Image layout:
https://github.com/ZYecho/image-spec/blob/master/image-layout.md
这个是说镜像体现在文件系统上的layout，当然可能是存在于tar包或者nfs等。

### 流程解析
    //snapshotterName default为overlay
    func (i *image) Unpack(ctx context.Context, snapshotterName string) error {
        // lease的意义？应该是和GC相关
    	ctx, done, err := i.client.WithLease(ctx)
    	if err != nil {
    		return err
    	}
    	defer done(ctx)
    
    	layers, err := i.getLayers(ctx, i.platform)
    	if err != nil {
    		return err
    	}
    
    	var (
    		sn = i.client.SnapshotService(snapshotterName)
    		a  = i.client.DiffService()
    		cs = i.client.ContentStore()
    
    		chain    []digest.Digest
    		unpacked bool
    	)
    	for _, layer := range layers {
    		unpacked, err = rootfs.ApplyLayer(ctx, layer, chain, sn, a)
    		if err != nil {
    			return err
    		}
            //unpack成功了
    		if unpacked {
    			// Set the uncompressed label after the uncompressed
    			// digest has been verified through apply.
    			// 看起来这个意思是会把compressed的变成uncompressed的？有待确认
    			cinfo := content.Info{
    				Digest: layer.Blob.Digest,
    				Labels: map[string]string{
    					"containerd.io/uncompressed": layer.Diff.Digest.String(),
    				},
    			}
    			if _, err := cs.Update(ctx, cinfo, "labels.containerd.io/uncompressed"); err != nil {
    				return err
    			}
    		}
    
    		chain = append(chain, layer.Diff.Digest)
    	}
        //最后一层也unpack成功了
    	if unpacked {
    		desc, err := i.i.Config(ctx, cs, i.platform)
    		if err != nil {
    			return err
    		}
    
    		rootfs := identity.ChainID(chain).String()
    
    		cinfo := content.Info{
    			Digest: desc.Digest,
    			Labels: map[string]string{
    				fmt.Sprintf("containerd.io/gc.ref.snapshot.%s", snapshotterName): rootfs,
    			},
    		}
    		if _, err := cs.Update(ctx, cinfo, fmt.Sprintf("labels.containerd.io/gc.ref.snapshot.%s", snapshotterName)); err != nil {
    			return err
    		}
    	}
    
    	return nil
    }
    

    func (i *image) getLayers(ctx context.Context, platform platforms.MatchComparer) ([]rootfs.Layer, error) {
    	cs := i.client.ContentStore()
        //从content中根据manifest的desc读出符合条件的Manifest对象
    	manifest, err := images.Manifest(ctx, cs, i.i.Target, platform)
    	if err != nil {
    		return nil, err
    	}
        //拿到image config里的diffids
    	diffIDs, err := i.i.RootFS(ctx, cs, platform)
    	if err != nil {
    		return nil, errors.Wrap(err, "failed to resolve rootfs")
    	}
    	// 查看镜像层数是否相同
    	if len(diffIDs) != len(manifest.Layers) {
    		return nil, errors.Errorf("mismatched image rootfs and manifest layers")
    	}
    	layers := make([]rootfs.Layer, len(diffIDs))
    	for i := range diffIDs {
    	    //这个是image Config的desc, ->  tar
    		layers[i].Diff = ocispec.Descriptor{
    			// TODO: derive media type from compressed type
    			MediaType: ocispec.MediaTypeImageLayer,
    			Digest:    diffIDs[i],
    		}
    		//这个是Manifest里边的desec, -> tar+gzip
    		layers[i].Blob = manifest.Layers[i]
    	}
    	return layers, nil
    }
    
path: containerd/containerd/rootfs/apply.go

    func applyLayers(ctx context.Context, layers []Layer, chain []digest.Digest, sn snapshots.Snapshotter, a diff.Applier, opts ...snapshots.Opt) error {
    	var (
    		parent  = identity.ChainID(chain[:len(chain)-1])
    		chainID = identity.ChainID(chain)
    		layer   = layers[len(layers)-1]
    		diff    ocispec.Descriptor
    		key     string
    		mounts  []mount.Mount
    		err     error
    	)
    
    	for {
    	    // 注意这个Key并不是完全等价于chainID
    		key = fmt.Sprintf("extract-%s %s", uniquePart(), chainID)
    
    		// Prepare snapshot with from parent, label as root
    		// step1 获取到经过COW之后的可挂载的mounts(type可能为bind brtfs等，目录和moby存储目录类似)
    		mounts, err = sn.Prepare(ctx, key, parent.String(), opts...)
    		if err != nil {
    			if errdefs.IsNotFound(err) && len(layers) > 1 {
    				if err := applyLayers(ctx, layers[:len(layers)-1], chain[:len(chain)-1], sn, a); err != nil {
    					if !errdefs.IsAlreadyExists(err) {
    						return err
    					}
    				}
    				// Do no try applying layers again
    				layers = nil
    				continue
    			} else if errdefs.IsAlreadyExists(err) {
    				// Try a different key
    				continue
    			}
    
    			// Already exists should have the caller retry
    			return errors.Wrapf(err, "failed to prepare extraction snapshot %q", key)
    
    		}
    		break
    	}
    	defer func() {
    		if err != nil {
    			if !errdefs.IsAlreadyExists(err) {
    				log.G(ctx).WithError(err).WithField("key", key).Infof("apply failure, attempting cleanup")
    			}
    
    			if rerr := sn.Remove(ctx, key); rerr != nil {
    				log.G(ctx).WithError(rerr).WithField("key", key).Warnf("extraction snapshot removal failed")
    			}
    		}
    	}()
        
        // step2, Blob依然是tar+gzip的形式
        // 先将mounts挂载到一个temp dir上并且applyDiff，要明白DIffs在mount的path原有挂载点目录里也是可见的，和temp dir在哪没有关系
    	diff, err = a.Apply(ctx, layer.Blob, mounts)
    	if err != nil {
    		err = errors.Wrapf(err, "failed to extract layer %s", layer.Diff.Digest)
    		return err
    	}
    	// 判断一下是否符合预期
    	if diff.Digest != layer.Diff.Digest {
    		err = errors.Errorf("wrong diff id calculated on extraction %q", diff.Digest)
    		return err
    	}
        //step3
    	if err = sn.Commit(ctx, chainID.String(), key, opts...); err != nil {
    		err = errors.Wrapf(err, "failed to commit snapshot %s", key)
    		return err
    	}
    
    	return nil
    }
    
    // 有兴趣可以跟一下apply的细节
    // Apply applies a tar stream of an OCI style diff tar.
    // See https://github.com/opencontainers/image-spec/blob/master/layer.md#applying-changesets
    func Apply(ctx context.Context, root string, r io.Reader, opts ...ApplyOpt) (int64, error) {
    	root = filepath.Clean(root)
    
    	var options ApplyOptions
    	for _, opt := range opts {
    		if err := opt(&options); err != nil {
    			return 0, errors.Wrap(err, "failed to apply option")
    		}
    	}
    	if options.Filter == nil {
    		options.Filter = all
    	}
    
    	return apply(ctx, root, tar.NewReader(r), options)
    }

这一节最后有关snapshotter的细节比较多，特别是step1-step3，会在介绍snapshotter和具体storage driver的时候看一下其工作机制和step1-3的细节，并总结回顾一下这部分内容。

  [1]: http://static.zybuluo.com/myecho/5ge7ybzdsb69920sxvkd958k/image.png
  [2]: http://static.zybuluo.com/myecho/75bv8w7hnh82usvhwe6kok02/image.png
