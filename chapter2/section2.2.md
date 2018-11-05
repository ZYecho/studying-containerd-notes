# Task API

首先看一下关于Task API都有哪些接口， path在 containerd/api/services/tasks/v1/tasks.pb.go，

    var _Tasks_serviceDesc = grpc.ServiceDesc{
    	ServiceName: "containerd.services.tasks.v1.Tasks",
    	HandlerType: (*TasksServer)(nil),
    	Methods: []grpc.MethodDesc{
    		{
    			MethodName: "Create",
    			Handler:    _Tasks_Create_Handler,
    		},
    		{
    			MethodName: "Start",
    			Handler:    _Tasks_Start_Handler,
    		},
    		{
    			MethodName: "Delete",
    			Handler:    _Tasks_Delete_Handler,
    		},
    		{
    			MethodName: "DeleteProcess",
    			Handler:    _Tasks_DeleteProcess_Handler,
    		},
    		{
    			MethodName: "Get",
    			Handler:    _Tasks_Get_Handler,
    		},
    		{
    			MethodName: "List",
    			Handler:    _Tasks_List_Handler,
    		},
    		{
    			MethodName: "Kill",
    			Handler:    _Tasks_Kill_Handler,
    		},
    		{
    			MethodName: "Exec",
    			Handler:    _Tasks_Exec_Handler,
    		},
    		{
    			MethodName: "ResizePty",
    			Handler:    _Tasks_ResizePty_Handler,
    		},
    		{  //关于容器I/O的部分我们后边会着重看一下
    			MethodName: "CloseIO",
    			Handler:    _Tasks_CloseIO_Handler,
    		},
    		{
    			MethodName: "Pause",
    			Handler:    _Tasks_Pause_Handler,
    		},
    		{
    			MethodName: "Resume",
    			Handler:    _Tasks_Resume_Handler,
    		},
    		{
    			MethodName: "ListPids",
    			Handler:    _Tasks_ListPids_Handler,
    		},
    		{
    			MethodName: "Checkpoint",
    			Handler:    _Tasks_Checkpoint_Handler,
    		},
    		{
    			MethodName: "Update",
    			Handler:    _Tasks_Update_Handler,
    		},
    		{
    			MethodName: "Metrics",
    			Handler:    _Tasks_Metrics_Handler,
    		},
    		{
    			MethodName: "Wait",
    			Handler:    _Tasks_Wait_Handler,
    		},
    	},
    	Streams:  []grpc.StreamDesc{},
    	Metadata: "github.com/containerd/containerd/api/services/tasks/v1/tasks.proto",
    }

来看一下run一个容器的逻辑链路。

path: containerd/cmd/ctr/commands/run/run.go
具体代码的就不截图了，而实现的主要逻辑如下所示:

    NewContainer -> c.ContainerService().Create(ctx, container)//创建container这个metadata
    //其中还会针对是否有无checkpoint来进行区别
    
    NewTask -> container.NewTask(ctx, ioCreator, opts...) -> c.client.TaskService().Create(ctx, request) //也验证了必须首先有container的metadata才能够创建，只有当Task被执行的时候才会创建一个真正的容器
    
    task.Start(ctx)
    
    //如果收到退出信号
    task.Delete(ctx) 
    //返回状态码


