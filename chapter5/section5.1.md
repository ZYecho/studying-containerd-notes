# native

native即用原生的linux文件系统比如ext4等完成snapshottor的工作，虽然在应用环境中很少使用，但对于实现自己的snapshottor有很好的借鉴意义，因为其细节比较简单，对于理解snapshottor的主要流程有很好的帮助。

我们来学习snapshottor主要放在prepare和commit两个方法上。
    
    // 这里的key应该是chaninID
    func (o *snapshotter) Prepare(ctx context.Context, key, parent string, opts ...snapshots.Opt) ([]mount.Mount, error) {
    	return o.createSnapshot(ctx, snapshots.KindActive, key, parent, opts)
    }
    
    func (o *snapshotter) createSnapshot(ctx context.Context, kind snapshots.Kind, key, parent string, opts []snapshots.Opt) ([]mount.Mount, error) {
    	var (
    		err      error
    		path, td string
    	)
    
    	if kind == snapshots.KindActive || parent == "" {
    	    // 如果是一个active的snapshot或者是no parent需要搞一个目录来存储这一层的内容(相当于moby的)RW层
    		td, err = ioutil.TempDir(filepath.Join(o.root, "snapshots"), "new-")
    		if err != nil {
    			return nil, errors.Wrap(err, "failed to create temp dir")
    		}
    		if err := os.Chmod(td, 0755); err != nil {
    			return nil, errors.Wrapf(err, "failed to chmod %s to 0755", td)
    		}
    		defer func() {
    			if err != nil {
    				if td != "" {
    					if err1 := os.RemoveAll(td); err1 != nil {
    						err = errors.Wrapf(err, "remove failed: %v", err1)
    					}
    				}
    				if path != "" {
    					if err1 := os.RemoveAll(path); err1 != nil {
    						err = errors.Wrapf(err, "failed to remove path: %v", err1)
    					}
    				}
    			}
    		}()
    	}
    
        // MetaStore开启一个事务
    	ctx, t, err := o.ms.TransactionContext(ctx, true)
    	if err != nil {
    		return nil, err
    	}
        //完成存储层元数据的管理
    	s, err := storage.CreateSnapshot(ctx, kind, key, parent, opts...)
    	if err != nil {
    		if rerr := t.Rollback(); rerr != nil {
    			log.G(ctx).WithError(rerr).Warn("failed to rollback transaction")
    		}
    		return nil, errors.Wrap(err, "failed to create snapshot")
    	}
        
        // 如果是需要创建新的目录的话
    	if td != "" {
    	    // layer have parent
    		if len(s.ParentIDs) > 0 {
    		    // 直接拿直系parent的即可，不支持多层的parent没有意义
    			parent := o.getSnapshotDir(s.ParentIDs[0])
    			// 因为native filesystem没有提供COW的能力，因此必须全量拷贝parent的目录
    			if err := fs.CopyDir(td, parent); err != nil {
    				return nil, errors.Wrap(err, "copying of parent failed")
    			}
    		}
            // s.ID应该每个snapshot唯一，不能冲突
    		path = o.getSnapshotDir(s.ID)
    		// 重命名临时目录，因此path对于snapshot应该是唯一的，和s.ID一一对应
    		if err := os.Rename(td, path); err != nil {
    			if rerr := t.Rollback(); rerr != nil {
    				log.G(ctx).WithError(rerr).Warn("failed to rollback transaction")
    			}
    			return nil, errors.Wrap(err, "failed to rename")
    		}
    		td = ""
    	}
    
    	if err := t.Commit(); err != nil {
    		return nil, errors.Wrap(err, "commit failed")
    	}
    
    	return o.mounts(s), nil
    }
    
    // 其实就是在bolt的元数据的存储中登记一下
    func (o *snapshotter) mounts(s storage.Snapshot) []mount.Mount {
    	var (
    		roFlag string
    		source string
    	)
        
        // 区分是否可读写
    	if s.Kind == snapshots.KindView {
    		roFlag = "ro"
    	} else {
    		roFlag = "rw"
    	}
    
    	if len(s.ParentIDs) == 0 || s.Kind == snapshots.KindActive {
    		source = o.getSnapshotDir(s.ID)
    	} else {
    	    // 只读的话直接拿parent的即可
    		source = o.getSnapshotDir(s.ParentIDs[0])
    	}
    
    	return []mount.Mount{
    		{
    			Source: source,
    			// 联合挂载，可在本机上实验一下
    			Type:   "bind",
    			Options: []string{
    				roFlag,
    				// rbind表示会把原来目录下的挂载点也会一起挂载过去
    				"rbind",
    			},
    		},
    	}
    }
    
    // 在bolt元数据存储中标记一下，表示snapshot为commited状态
    func (o *snapshotter) Commit(ctx context.Context, name, key string, opts ...snapshots.Opt) error {
    	ctx, t, err := o.ms.TransactionContext(ctx, true)
    	if err != nil {
    		return err
    	}
    
    	id, _, _, err := storage.GetInfo(ctx, key)
    	if err != nil {
    		return err
    	}
    
    	usage, err := fs.DiskUsage(ctx, o.getSnapshotDir(id))
    	if err != nil {
    		return err
    	}
    
    	if _, err := storage.CommitActive(ctx, key, name, snapshots.Usage(usage), opts...); err != nil {
    		if rerr := t.Rollback(); rerr != nil {
    			log.G(ctx).WithError(rerr).Warn("failed to rollback transaction")
    		}
    		return errors.Wrap(err, "failed to commit snapshot")
    	}
    	return t.Commit()
    }
