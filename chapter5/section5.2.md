# overlayfs


> OverlayFS is a modern union filesystem that is similar to AUFS, but
> faster and with a simpler implementation. Docker provides two storage
> drivers for OverlayFS: the original overlay, and the newer and more
> stable overlay2.

### 预备知识
https://docs.docker.com/storage/storagedriver/overlayfs-driver/

overlay vs overlay2
> If you are still using the overlay driver rather than overlay2, see
> How the overlay driver works instead.
> 
> OverlayFS layers two directories on a single Linux host and presents
> them as a single directory. These directories are called layers and
> the unification process is referred to as a union mount. OverlayFS
> refers to the lower directory as lowerdir and the upper directory a
> upperdir. The unified view is exposed through its own directory called
> merged.
> 
> While the overlay driver only works with a single lower OverlayFS
> layer and hence requires hard links for implementation of
> multi-layered images, the overlay2 driver natively supports up to 128
> lower OverlayFS layers. This capability provides better performance
> for layer-related Docker commands such as docker build and docker
> commit, and consumes fewer inodes on the backing filesystem.

overlay1存在的问题:

> While the overlay driver only works with a single lower OverlayFS
> layer and hence requires hard links for implementation of
> multi-layered images, the overlay2 driver natively supports up to 128
> lower OverlayFS layers. This capability provides better performance
> for layer-related Docker commands such as docker build and docker
> commit, and consumes fewer inodes on the backing filesystem.

如果还是在同一个文件系统的时候，比如pull镜像的时候是可以使用hard-link的方式来引用低层的镜像层的数据的。

    To create a container, the overlay driver combines the directory representing the image’s top layer plus a new directory for the container. The image’s top layer is the lowerdir in the overlay and is read-only. The new directory for the container is the upperdir and is writable.

moby overlay的代码：moby/moby/daemon/graphdriver/overlay
* `Create()`方法也就是产生rootfs的过程中都是`copy.DirCopy(parentUpperDir, upperDir, copy.Content, true)`也就是拷贝的内容(将parentUpperDir的内容拷贝到upperDir,因为只支持一层lowerDir，同时不能将parentUpperDir hard-link到upperDir目录，因为upper层是可读写的，hard-link也会破坏原有的parent)
* `ApplyDiff()`也就是构建镜像过程中调用的函数都是使用的hard-link的方式`copy.DirCopy(parentRootDir, tmpRootDir, copy.Hardlink, true)`

docker commit过程需要`Create() -> Diff() -> ApplyDiff()`的过程，因为`Create()`会拷贝文件内容，因此会消耗多余的inode。

### 具体实现

    func (o *snapshotter) Prepare(ctx context.Context, key, parent string, opts ...snapshots.Opt) ([]mount.Mount, error) {
    	return o.createSnapshot(ctx, snapshots.KindActive, key, parent, opts)
    }
    
    func (o *snapshotter) createSnapshot(ctx context.Context, kind snapshots.Kind, key, parent string, opts []snapshots.Opt) ([]mount.Mount, error) {
    	ctx, t, err := o.ms.TransactionContext(ctx, true)
    	if err != nil {
    		return nil, err
    	}
    
    	var td, path string
    	defer func() {
    		if err != nil {
    			if td != "" {
    				if err1 := os.RemoveAll(td); err1 != nil {
    					log.G(ctx).WithError(err1).Warn("failed to cleanup temp snapshot directory")
    				}
    			}
    			if path != "" {
    				if err1 := os.RemoveAll(path); err1 != nil {
    					log.G(ctx).WithError(err1).WithField("path", path).Error("failed to reclaim snapshot directory, directory may need removal")
    					err = errors.Wrapf(err, "failed to remove path: %v", err1)
    				}
    			}
    		}
    	}()
    
    	snapshotDir := filepath.Join(o.root, "snapshots")
    	td, err = o.prepareDirectory(ctx, snapshotDir, kind)
    	if err != nil {
    		if rerr := t.Rollback(); rerr != nil {
    			log.G(ctx).WithError(rerr).Warn("failed to rollback transaction")
    		}
    		return nil, errors.Wrap(err, "failed to create prepare snapshot dir")
    	}
    	rollback := true
    	defer func() {
    		if rollback {
    			if rerr := t.Rollback(); rerr != nil {
    				log.G(ctx).WithError(rerr).Warn("failed to rollback transaction")
    			}
    		}
    	}()
    
    	s, err := storage.CreateSnapshot(ctx, kind, key, parent, opts...)
    	if err != nil {
    		return nil, errors.Wrap(err, "failed to create snapshot")
    	}
    
    	if len(s.ParentIDs) > 0 {
    		st, err := os.Stat(o.upperPath(s.ParentIDs[0]))
    		if err != nil {
    			return nil, errors.Wrap(err, "failed to stat parent")
    		}
    
    		stat := st.Sys().(*syscall.Stat_t)
            
            // 设置目录权限和parent的相同
    		if err := os.Lchown(filepath.Join(td, "fs"), int(stat.Uid), int(stat.Gid)); err != nil {
    			if rerr := t.Rollback(); rerr != nil {
    				log.G(ctx).WithError(rerr).Warn("failed to rollback transaction")
    			}
    			return nil, errors.Wrap(err, "failed to chown")
    		}
    	}
    
    	path = filepath.Join(snapshotDir, s.ID)
    	if err = os.Rename(td, path); err != nil {
    		return nil, errors.Wrap(err, "failed to rename")
    	}
    	td = ""
        
        // 这里不太理解为什么rollback=false? 
    	rollback = false
    	if err = t.Commit(); err != nil {
    		return nil, errors.Wrap(err, "commit failed")
    	}
        
        // merge目录不由snapshotter决定，挂载在哪，哪就是merge层
    	return o.mounts(s), nil
    }

    func (o *snapshotter) mounts(s storage.Snapshot) []mount.Mount {
    	if len(s.ParentIDs) == 0 {
    		// if we only have one layer/no parents then just return a bind mount as overlay
    		// will not work
    		roFlag := "rw"
    		if s.Kind == snapshots.KindView {
    			roFlag = "ro"
    		}
    
    		return []mount.Mount{
    			{
    				Source: o.upperPath(s.ID),
    				Type:   "bind",
    				Options: []string{
    					roFlag,
    					"rbind",
    				},
    			},
    		}
    	}
    	var options []string
    
    	if s.Kind == snapshots.KindActive {
    		options = append(options,
    		    // filepath.Join(o.root, "snapshots", id, "work")
    		    // 文件系统挂载后用于存放临时和间接文件的工作基目录
    			fmt.Sprintf("workdir=%s", o.workPath(s.ID)),
    			// upper目录，也就是文件系统存储的主目录，可以认为是container的RW层
    			// filepath.Join(o.root, "snapshots", id, "fs")
    			fmt.Sprintf("upperdir=%s", o.upperPath(s.ID)),
    		)
    	} else if len(s.ParentIDs) == 1 {
    	    // 只有一层且只是返回可读层的时候上边注释也有说明，直接返回父亲的bind mount即可
    		return []mount.Mount{
    			{
    				Source: o.upperPath(s.ParentIDs[0]),
    				Type:   "bind",
    				Options: []string{
    					"ro",
    					"rbind",
    				},
    			},
    		}
    	}
        
        // 不论是返回读写层都需要把lowerdir给放到options中去
    	parentPaths := make([]string, len(s.ParentIDs))
    	for i := range s.ParentIDs {
    		parentPaths[i] = o.upperPath(s.ParentIDs[i])
    	}
    
    	options = append(options, fmt.Sprintf("lowerdir=%s", strings.Join(parentPaths, ":")))
    	// 具体如何处理由mount包处理，参考containerd/containerd/sys/mount_linux.go
    	return []mount.Mount{
    		{
    			Type:    "overlay",
    			Source:  "overlay",
    			Options: options,
    		},
    	}
    
    }
    
    // 和native的一样的，没什么好说的
    func (o *snapshotter) Commit(ctx context.Context, name, key string, opts ...snapshots.Opt) error {
    	ctx, t, err := o.ms.TransactionContext(ctx, true)
    	if err != nil {
    		return err
    	}
    
    	defer func() {
    		if err != nil {
    			if rerr := t.Rollback(); rerr != nil {
    				log.G(ctx).WithError(rerr).Warn("failed to rollback transaction")
    			}
    		}
    	}()
    
    	// grab the existing id
    	id, _, _, err := storage.GetInfo(ctx, key)
    	if err != nil {
    		return err
    	}
    
    	usage, err := fs.DiskUsage(ctx, o.upperPath(id))
    	if err != nil {
    		return err
    	}
    
    	if _, err = storage.CommitActive(ctx, key, name, snapshots.Usage(usage), opts...); err != nil {
    		return errors.Wrap(err, "failed to commit snapshot")
    	}
    	return t.Commit()
    }


