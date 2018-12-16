# btrfs


### 预备知识
对于Docker 社区版本来说，不同linux发行版的选择如下：
![不同系统支持][1]
对于不同的文件系统，推荐如下：
![文件系统所需要支持][2]

https://docs.docker.com/storage/storagedriver/btrfs-driver/

> Btrfs is a next generation copy-on-write filesystem that supports many
> advanced storage technologies that make it a good fit for Docker.
> Btrfs is included in the mainline Linux kernel.
> btrfs requires a dedicated block storage device such as a physical
> disk. This block device must be formatted for Btrfs and mounted into
> /var/lib/docker/.
> 
> One of the benefits of Btrfs is the ease of managing Btrfs filesystems
> without the need to unmount the filesystem or restart Docker.
> 
> When space gets low, Btrfs automatically expands the volume in chunks
> of roughly 1 GB.
> 
> To add a block device to a Btrfs volume, use the btrfs device add and
> btrfs filesystem balance commands.
> 
>     $ sudo btrfs device add /dev/svdh /var/lib/docker
>     $ sudo btrfs filesystem balance /var/lib/docker
> 
> With Btrfs, writing and updating lots of small files can result in
> slow performance.

适合大I/O的场景，且由于其日志的实现，顺序写的性能也不会很高。

### 具体实现

    func (b *snapshotter) Prepare(ctx context.Context, key, parent string, opts ...snapshots.Opt) ([]mount.Mount, error) {
    	return b.makeSnapshot(ctx, snapshots.KindActive, key, parent, opts)
    }
    
    func (b *snapshotter) makeSnapshot(ctx context.Context, kind snapshots.Kind, key, parent string, opts []snapshots.Opt) ([]mount.Mount, error) {
    	ctx, t, err := b.ms.TransactionContext(ctx, true)
    	if err != nil {
    		return nil, err
    	}
    	defer func() {
    		if err != nil && t != nil {
    			if rerr := t.Rollback(); rerr != nil {
    				log.G(ctx).WithError(rerr).Warn("failed to rollback transaction")
    			}
    		}
    	}()
    
    	s, err := storage.CreateSnapshot(ctx, kind, key, parent, opts...)
    	if err != nil {
    		return nil, err
    	}
        // 形如root/active/snapshotId
        // 这里为什么要加kind？不能直接全部放到snapshots里边去？
    	target := filepath.Join(b.root, strings.ToLower(s.Kind.String()), s.ID)
    
    	if len(s.ParentIDs) == 0 {
    		// create new subvolume
    		// btrfs subvolume create /dir
    		// 不做快照直接创建subvolume
    		if err = btrfs.SubvolCreate(target); err != nil {
    			return nil, err
    		}
    	} else {
    	    // root/snapshots/parent (单亲关系，不存在多个父节点的情况)
    	    /*
    	    var (
		        active    = filepath.Join(root, "active")
		        view      = filepath.Join(root, "view")
		        snapshots = filepath.Join(root, "snapshots")
	        )
    	    */
    		parentp := filepath.Join(b.root, "snapshots", s.ParentIDs[0])
    
    		var readonly bool
    		if kind == snapshots.KindView {
    			readonly = true
    		}
    
    		// btrfs subvolume snapshot /parent /subvol
    		// 通过snapshot进行创建是因为其底层共享storage pool的数据，符合COW的语义
    		if err = btrfs.SubvolSnapshot(target, parentp, readonly); err != nil {
    			return nil, err
    		}
    	}
    	err = t.Commit()
    	t = nil
    	if err != nil {
    		if derr := btrfs.SubvolDelete(target); derr != nil {
    			log.G(ctx).WithError(derr).WithField("subvolume", target).Error("failed to delete subvolume")
    		}
    		return nil, err
    	}
    
    	return b.mounts(target, s)
    }
    
    func (b *snapshotter) mounts(dir string, s storage.Snapshot) ([]mount.Mount, error) {
    	var options []string
    
    	// get the subvolume id back out for the mount
    	sid, err := btrfs.SubvolID(dir)
    	if err != nil {
    		return nil, err
    	}
    
    	options = append(options, fmt.Sprintf("subvolid=%d", sid))
    
    	if s.Kind != snapshots.KindActive {
    		options = append(options, "ro")
    	}
    
    	return []mount.Mount{
    		{
    			Type:   "btrfs",
    			Source: b.device,
    			// NOTE(stevvooe): While it would be nice to use to uuids for
    			// mounts, they don't work reliably if the uuids are missing.
    			Options: options,
    		},
    	}, nil
    }

    // 这里和前边简介的commit操作有所不同，不是只做了元数据的处理
    func (b *snapshotter) Commit(ctx context.Context, name, key string, opts ...snapshots.Opt) (err error) {
    	usage, err := b.usage(ctx, key)
    	if err != nil {
    		return errors.Wrap(err, "failed to compute usage")
    	}
    
    	ctx, t, err := b.ms.TransactionContext(ctx, true)
    	if err != nil {
    		return err
    	}
    	defer func() {
    		if err != nil && t != nil {
    			if rerr := t.Rollback(); rerr != nil {
    				log.G(ctx).WithError(rerr).Warn("failed to rollback transaction")
    			}
    		}
    	}()
    
    	id, err := storage.CommitActive(ctx, key, name, usage, opts...) // TODO(stevvooe): Resolve a usage value for btrfs
    	if err != nil {
    		return errors.Wrap(err, "failed to commit")
    	}
    
    	source := filepath.Join(b.root, "active", id)
    	target := filepath.Join(b.root, "snapshots", id)
        
        // 必须完成active到可读写的snapshot的转化，因为前边prepare用的是snapshots的目录
        // commit的时候是 active -> snapshot
        // prepare的时候是 parent snapshot -> snapshot的转换，保证parent snapshot是只读的
    	if err := btrfs.SubvolSnapshot(target, source, true); err != nil {
    		return err
    	}
    
    	err = t.Commit()
    	t = nil
    	if err != nil {
    		if derr := btrfs.SubvolDelete(target); derr != nil {
    			log.G(ctx).WithError(derr).WithField("subvolume", target).Error("failed to delete subvolume")
    		}
    		return err
    	}
    
    	if derr := btrfs.SubvolDelete(source); derr != nil {
    		// Log as warning, only needed for cleanup, will not cause name collision
    		log.G(ctx).WithError(derr).WithField("subvolume", source).Warn("failed to delete subvolume")
    	}
    
    	return nil
    }

  [1]: https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/20180309151325-linux-distribution.jpg
  [2]: https://raw.githubusercontent.com/zhangchenchen/zhangchenchen.github.io/hexo/images/20180309151640-file-system.jpg

