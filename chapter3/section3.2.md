# Container I/O

那么在整个运作过程容器的I/O是如何通过containerd的API来操作的呢？I/O如何被打开？I/O如何被关闭？I/O之间如何进行数据交换的操作，都将在这一节得到答案。

path: containerd/cmd/ctr/commands/tasks/tasks_unix.go

    // NewTask creates a new task
    func NewTask(ctx gocontext.Context, client *containerd.Client, container containerd.Container, checkpoint string, con console.Console, nullIO bool, ioOpts []cio.Opt, opts ...containerd.NewTaskOpts) (containerd.Task, error) {
        //默认是使用stdin来初始化client的I/O，如os.stdout/os.stdin/os.stderr
    	stdio := cio.NewCreator(append([]cio.Opt{cio.WithStdio}, ioOpts...)...)
    	if checkpoint != "" {
    		im, err := client.GetImage(ctx, checkpoint)
    		if err != nil {
    			return nil, err
    		}
    		opts = append(opts, containerd.WithTaskCheckpoint(im))
    	}
    	ioCreator := stdio
    	// 如果指定了tty
    	if con != nil {
    	    // 改变client端的stdion,  stdin -> console, stdout -> console, stderr->nil
    		ioCreator = cio.NewCreator(append([]cio.Opt{cio.WithStreams(con, con, nil), cio.WithTerminal}, ioOpts...)...)
    	}
    	if nullIO {
    		if con != nil {
    			return nil, errors.New("tty and null-io cannot be used together")
    		}
    		ioCreator = cio.NullIO
    	}
    	//最终会调用containerd的taskService, 这里创建的FIFO-DIR也会作为参数传入
    	return container.NewTask(ctx, ioCreator, opts...)
    }
    
    // NewCreator returns an IO creator from the options
    func NewCreator(opts ...Opt) Creator {
    	streams := &Streams{}
    	for _, opt := range opts {
    		opt(streams)
    	}
    	if streams.FIFODir == "" {
    		streams.FIFODir = defaults.DefaultFIFODir
    	}
    	return func(id string) (IO, error) {
    	    //创建containerd对外的I/O管道，所有的I/O都是通过fifo与client端的I/O进行数据交换的
    	    // path形如:filepath.Join(dir, id+"-stdin"), id就是container id
    		fifos, err := NewFIFOSetInDir(streams.FIFODir, id, streams.Terminal)
    		if err != nil {
    			return nil, err
    		}
    		if streams.Stdin == nil {
    			fifos.Stdin = ""
    		}
    		if streams.Stdout == nil {
    			fifos.Stdout = ""
    		}
    		if streams.Stderr == nil {
    			fifos.Stderr = ""
    		}
    		//真正的数据交换过程
    		return copyIO(fifos, streams)
    	}
    }
    
    //ioset对应的就是containerd client端的I/O
    func copyIO(fifos *FIFOSet, ioset *Streams) (*cio, error) {
    	var ctx, cancel = context.WithCancel(context.Background())
    	//注重看下打开io的方式
    	pipes, err := openFifos(ctx, fifos)
    	if err != nil {
    		cancel()
    		return nil, err
    	}
    
    	if fifos.Stdin != "" {
    		go func() {
    			p := bufPool.Get().(*[]byte)
    			defer bufPool.Put(p)
    
    			io.CopyBuffer(pipes.Stdin, ioset.Stdin, *p)
    			pipes.Stdin.Close()
    		}()
    	}
    
    	var wg = &sync.WaitGroup{}
    	wg.Add(1)
    	go func() {
    		p := bufPool.Get().(*[]byte)
    		defer bufPool.Put(p)
    
    		io.CopyBuffer(ioset.Stdout, pipes.Stdout, *p)
    		pipes.Stdout.Close()
    		wg.Done()
    	}()
        
        //对于terminal的环境来说，stdout/stderr统一拷贝到stdout，两者不区分
    	if !fifos.Terminal {
    		wg.Add(1)
    		go func() {
    			p := bufPool.Get().(*[]byte)
    			defer bufPool.Put(p)
    
    			io.CopyBuffer(ioset.Stderr, pipes.Stderr, *p)
    			pipes.Stderr.Close()
    			wg.Done()
    		}()
    	}
    	return &cio{
    		config:  fifos.Config,
    		wg:      wg,
    		closers: append(pipes.closers(), fifos),
    		cancel:  cancel,
    	}, nil
    }
    
    func openFifos(ctx context.Context, fifos *FIFOSet) (pipes, error) {
    	var err error
    	defer func() {
    		if err != nil {
    			fifos.Close()
    		}
    	}()
    
    	var f pipes
    	if fifos.Stdin != "" {
    	    //非堵塞方式打开写端
    		if f.Stdin, err = fifo.OpenFifo(ctx, fifos.Stdin, syscall.O_WRONLY|syscall.O_CREAT|syscall.O_NONBLOCK, 0700); err != nil {
    			return f, errors.Wrapf(err, "failed to open stdin fifo")
    		}
    		defer func() {
    			if err != nil && f.Stdin != nil {
    				f.Stdin.Close()
    			}
    		}()
    	}
    	if fifos.Stdout != "" {
    	    //非堵塞方式打开读端
    		if f.Stdout, err = fifo.OpenFifo(ctx, fifos.Stdout, syscall.O_RDONLY|syscall.O_CREAT|syscall.O_NONBLOCK, 0700); err != nil {
    			return f, errors.Wrapf(err, "failed to open stdout fifo")
    		}
    		defer func() {
    			if err != nil && f.Stdout != nil {
    				f.Stdout.Close()
    			}
    		}()
    	}
    	if fifos.Stderr != "" {
    	    //非堵塞方式打开读端
    		if f.Stderr, err = fifo.OpenFifo(ctx, fifos.Stderr, syscall.O_RDONLY|syscall.O_CREAT|syscall.O_NONBLOCK, 0700); err != nil {
    			return f, errors.Wrapf(err, "failed to open stderr fifo")
    		}
    	}
    	return f, nil
    }

对于container的daemon来说，它只需要将I/O通过fifo管道写出去即可(其实还是通过底层runc和shim交互来完成的)，不需要关心谁从FIFO管道消费。

接下来再看下，containerd daemon拿到fifo-dir后会如何处理:

path: containerd/services/tasks/local.go

    opts := runtime.CreateOpts{
    		Spec: container.Spec,
    		//这里的I/O还是fifo-dir目录下的fifo
    		IO: runtime.IO{
    			Stdin:    r.Stdin,
    			Stdout:   r.Stdout,
    			Stderr:   r.Stderr,
    			Terminal: r.Terminal,
    		},
    		Checkpoint:     checkpointPath,
    		Runtime:        container.Runtime.Name,
    		RuntimeOptions: container.Runtime.Options,
    		TaskOptions:    r.Options,
    }
    c, err := runtime.Create(ctx, r.ContainerID, opts)

path: containerd/runtime/v1/linux/runtime.go

    sopts := &shim.CreateTaskRequest{
    		ID:         id,
    		Bundle:     bundle.path,
    		Runtime:    rt,
    		Stdin:      opts.IO.Stdin,
    		Stdout:     opts.IO.Stdout,
    		Stderr:     opts.IO.Stderr,
    		Terminal:   opts.IO.Terminal,
    		Checkpoint: opts.Checkpoint,
    		Options:    opts.TaskOptions,
    }
    cr, err := s.Create(ctx, sopts)
    //封装成task请求然后又交给shim进程来处理了

path: containerd/runtime/v1/shim/service.go

    if err := process.Create(ctx, config); err != nil {
    		return nil, errdefs.ToGRPC(err)
    }
    
path: containerd/runtime/v1/linux/proc/init.go

    // Create the process with the provided config
    func (p *Init) Create(ctx context.Context, r *CreateConfig) error {
    	var (
    		err    error
    		socket *runc.Socket
    	)
    	//根据是否tty有两种区分，也能体现在后边copyI/O的步骤上
    	if r.Terminal {
    		if socket, err = runc.NewTempConsoleSocket(); err != nil {
    			return errors.Wrap(err, "failed to create OCI runtime console socket")
    		}
    		defer socket.Close()
    	} else if hasNoIO(r) {
    		if p.io, err = runc.NewNullIO(); err != nil {
    			return errors.Wrap(err, "creating new NULL IO")
    		}
    	} else {
    	//这里是创建与容器物理进程通信的Pipe, 不是前边介绍的fifo-dir目录下的fifo
    		if p.io, err = runc.NewPipeIO(p.IoUID, p.IoGID, withConditionalIO(p.stdio)); err != nil {
    			return errors.Wrap(err, "failed to create OCI runtime io pipes")
    		}
    	}
    	pidFile := filepath.Join(p.Bundle, InitPidFile)
    	if r.Checkpoint != "" {
    		opts := &runc.RestoreOpts{
    			CheckpointOpts: runc.CheckpointOpts{
    				ImagePath:  r.Checkpoint,
    				WorkDir:    p.WorkDir,
    				ParentPath: r.ParentCheckpoint,
    			},
    			PidFile:     pidFile,
    			IO:          p.io,
    			NoPivot:     p.NoPivotRoot,
    			Detach:      true,
    			NoSubreaper: true,
    		}
    		p.initState = &createdCheckpointState{
    			p:    p,
    			opts: opts,
    		}
    		return nil
    	}
    	opts := &runc.CreateOpts{
    		PidFile:      pidFile,
    		IO:           p.io,
    		NoPivot:      p.NoPivotRoot,
    		NoNewKeyring: p.NoNewKeyring,
    	}
    	if socket != nil {
    		opts.ConsoleSocket = socket
    	}
    	//这是真正创建物理容器进程的步骤，在其中通过 opts.Set(cmd)将pipe I/O set进去
    	if err := p.runtime.Create(ctx, r.ID, r.Bundle, opts); err != nil {
    		return p.runtimeError(err, "OCI runtime create failed")
    	}
    	//这里为什么打开r.stdin的fifo写端，在CloseIO的时候关闭的就是它
    	//如果只有读端，如果外部进程异常退出了，那么容器只能认为容器输入结束了(因为对于fifo来说，唯一的写关闭了，读就会自动关闭)
    	//而如果增加了写端，就能够区分正常和异常结束，同时可以将关闭的主动权握在自己手里，正常结束通过closeIO来关闭写端来结束
    	if r.Stdin != "" {
    		sc, err := fifo.OpenFifo(ctx, r.Stdin, syscall.O_WRONLY|syscall.O_NONBLOCK, 0)
    		if err != nil {
    			return errors.Wrapf(err, "failed to open stdin fifo %s", r.Stdin)
    		}
    		p.stdin = sc
    		p.closers = append(p.closers, sc)
    	}
    	var copyWaitGroup sync.WaitGroup
    	//下边的逻辑是处理从pipe拷贝到fifo的过程
    	if socket != nil {
    		console, err := socket.ReceiveMaster()
    		if err != nil {
    			return errors.Wrap(err, "failed to retrieve console master")
    		}
    		console, err = p.Platform.CopyConsole(ctx, console, r.Stdin, r.Stdout, r.Stderr, &p.wg, &copyWaitGroup)
    		if err != nil {
    			return errors.Wrap(err, "failed to start console copy")
    		}
    		p.console = console
    	} else if !hasNoIO(r) {
    		if err := copyPipes(ctx, p.io, r.Stdin, r.Stdout, r.Stderr, &p.wg, &copyWaitGroup); err != nil {
    			return errors.Wrap(err, "failed to start io pipe copy")
    		}
    	}
    
    	copyWaitGroup.Wait()
    	pid, err := runc.ReadPidFile(pidFile)
    	if err != nil {
    		return errors.Wrap(err, "failed to retrieve OCI runtime container pid")
    	}
    	p.pid = pid
    	return nil
    }
    
    //以非tty的情况下举例子
    
    func copyPipes(ctx context.Context, rio runc.IO, stdin, stdout, stderr string, wg, cwg *sync.WaitGroup) error {
    	var sameFile io.WriteCloser
    	for _, i := range []struct {
    		name string
    		dest func(wc io.WriteCloser, rc io.Closer)
    	}{
    		{
    			name: stdout,
    			dest: func(wc io.WriteCloser, rc io.Closer) {
    				wg.Add(1)
    				cwg.Add(1)
    				go func() {
    					cwg.Done()
    					p := bufPool.Get().(*[]byte)
    					defer bufPool.Put(p)
    					io.CopyBuffer(wc, rio.Stdout(), *p)
    					wg.Done()
    					wc.Close()
    					if rc != nil {
    						rc.Close()
    					}
    				}()
    			},
    		}, {
    			name: stderr,
    			dest: func(wc io.WriteCloser, rc io.Closer) {
    				wg.Add(1)
    				cwg.Add(1)
    				go func() {
    					cwg.Done()
    					p := bufPool.Get().(*[]byte)
    					defer bufPool.Put(p)
    					io.CopyBuffer(wc, rio.Stderr(), *p)
    					wg.Done()
    					wc.Close()
    					if rc != nil {
    						rc.Close()
    					}
    				}()
    			},
    		},
    	} {//开始对stdout和stderr处理
    		ok, err := isFifo(i.name)
    		if err != nil {
    			return err
    		}
    		var (
    			fw io.WriteCloser
    			fr io.Closer
    		)
    		if ok {
    		    //为什么stdout和stderr读端和写端都打开？这里还是为了防止fifo的自动关闭，开了写再开个读，什么时候关闭全由自己决定
    			if fw, err = fifo.OpenFifo(ctx, i.name, syscall.O_WRONLY, 0); err != nil {
    				return fmt.Errorf("containerd-shim: opening %s failed: %s", i.name, err)
    			}
    			if fr, err = fifo.OpenFifo(ctx, i.name, syscall.O_RDONLY, 0); err != nil {
    				return fmt.Errorf("containerd-shim: opening %s failed: %s", i.name, err)
    			}
    		} else {
    			if sameFile != nil {
    				i.dest(sameFile, nil)
    				continue
    			}
    			if fw, err = os.OpenFile(i.name, syscall.O_WRONLY|syscall.O_APPEND, 0); err != nil {
    				return fmt.Errorf("containerd-shim: opening %s failed: %s", i.name, err)
    			}
    			if stdout == stderr {
    				sameFile = fw
    			}
    		}
    		i.dest(fw, fr)
    	}
    	if stdin == "" {
    		return nil
    	}
    	//stdin打开读端用作正常使用
    	f, err := fifo.OpenFifo(ctx, stdin, syscall.O_RDONLY|syscall.O_NONBLOCK, 0)
    	if err != nil {
    		return fmt.Errorf("containerd-shim: opening %s failed: %s", stdin, err)
    	}
    	cwg.Add(1)
    	go func() {
    		cwg.Done()
    		p := bufPool.Get().(*[]byte)
    		defer bufPool.Put(p)
    
    		io.CopyBuffer(rio.Stdin(), f, *p)
    		rio.Stdin().Close()
    		f.Close()
    	}()
    	return nil
    }

#### Container I/O的关闭
下边关注一下`_Tasks_CloseIO_Handler`，也就是处理I/O关闭的handler.

path: containerd/services/tasks/local.go

    func (l *local) CloseIO(ctx context.Context, r *api.CloseIORequest, _ ...grpc.CallOption) (*ptypes.Empty, error) {
    	t, err := l.getTask(ctx, r.ContainerID)
    	if err != nil {
    		return nil, err
    	}
    	p := runtime.Process(t)
    	if r.ExecID != "" {
    		if p, err = t.Process(ctx, r.ExecID); err != nil {
    			return nil, errdefs.ToGRPC(err)
    		}
    	}
    	if r.Stdin {
    		if err := p.CloseIO(ctx); err != nil {
    			return nil, err
    		}
    	}
    	return empty, nil
    }

    // CloseIO closes the provided IO pipe for the process
    func (p *process) CloseIO(ctx context.Context) error {
    	_, err := p.shim.task.CloseIO(ctx, &task.CloseIORequest{
    		ID:     p.shim.ID(),
    		ExecID: p.id,
    		Stdin:  true,
    	})
    	if err != nil {
    		return errdefs.FromGRPC(err)
    	}
    	return nil
    }
    
    // CloseIO of a process
    func (s *Service) CloseIO(ctx context.Context, r *shimapi.CloseIORequest) (*ptypes.Empty, error) {
    	s.mu.Lock()
    	defer s.mu.Unlock()
    	p := s.processes[r.ID]
    	if p == nil {
    		return nil, errdefs.ToGRPCf(errdefs.ErrNotFound, "process does not exist %s", r.ID)
    	}
    	//在这里看到其实是关闭了stdin(这里对应上文中打开的r.stdin的写端)
    	if stdin := p.Stdin(); stdin != nil {
    		if err := stdin.Close(); err != nil {
    			return nil, errors.Wrap(err, "close stdin")
    		}
    	}
    	return empty, nil
    }
    

### 总结
tty情况下，runc和shim之间通过unix socket进行通信， 通过runc create --console-socket value 可以看出，此时stdout和stderr不区分，统一走socket; 而当non-tty的情况下，是通过父子进程之间的pipe进行通信的, stdout和stderr是有区分的。

### 总结-流程图
![angular.js.png-8.9kB][1]


### Pouch I/O设计
![image.png-79.5kB][2]

https://github.com/alibaba/pouch/pull/2375/files

着重注意一下对这段话的理解:

> The contained-shim will open fifo twice for reading and writing. For
> the writing mode, the shim doesn't close stdin fifo until the client
> calls CloseIO. In some case, the pouch daemon might be crash before
> finishing the input. If shim doesn't hold writing mode fifo, the
> process in container will consider that it is EOF signal and exit.
> 
> Based on this case, if the client sends EOF signal to input channel,
> the pouchd should send the CloseIO to shim to let the process exit.
> 
> StdinOnce in container's config is used by attach request. If the
> StdinOnce is true, when one attach request finishes stream copy, the
> pouchd will closes the input of process. So other attach requests to
> the same container will be stopped.
> 
> If the user wants StdinOnce, it should set it to true during creating
> container.

Thanks to @fuwid explaination.

### 参考
http://hustcat.github.io/terminal-and-docker/
https://github.com/alibaba/pouch/pull/2375/files


  [1]: http://static.zybuluo.com/myecho/3xdwajlta9567m0bcjq77qc6/angular.js.png
  [2]: http://static.zybuluo.com/myecho/lig45a4gvcubxul5b9bqv05d/image.png
