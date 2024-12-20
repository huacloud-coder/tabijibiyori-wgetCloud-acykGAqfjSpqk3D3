
## 一：背景


### 1\. 讲故事


为什么说这东西比较坑人呢？是因为最近一个月接到了两个dump，都反应程序卡死无响应，最后分析下来是因为`线程饥饿`导致，那什么原因导致的线程饥饿呢？进一步分析发现罪魁祸首是 `MySql.Data`，这就让人无语了，并且反馈都是升级了`MySql.Data`驱动引发，接下来我们简单聊一下。


## 二： MySql.Data 到底怎么了


### 1\. 祸根溯源


早期版本的 `MySql.Data` 访问数据库都是以同步的方式进行，比如：`ExecuteReader` 而不是 `ExecuteReaderAsync`，随着项目的升级改造需要提升MySql.Data的版本， MySql为了向前兼容保留了同步方法，下面引用最新的 MySql.Data 9\.1\.0 截图和参考代码如下：


![](https://img2024.cnblogs.com/blog/214741/202412/214741-20241220122814059-88699629.png)



```

// MySql.Data, Version=9.1.0.0, Culture=neutral, PublicKeyToken=c5687fc88969c44d
// MySql.Data.MySqlClient.MySqlConnection
using System.Threading;

public override void Open()
{
	OpenAsync(execAsync: false, CancellationToken.None).GetAwaiter().GetResult();
}


// MySql.Data, Version=9.1.0.0, Culture=neutral, PublicKeyToken=c5687fc88969c44d
// MySql.Data.MySqlClient.MySqlCommand
using System.Data;
using System.Threading;

public new MySqlDataReader ExecuteReader()
{
	return ExecuteReaderAsync(CommandBehavior.Default, execAsync: false, CancellationToken.None).GetAwaiter().GetResult();
}

public override object ExecuteScalar()
{
	return ExecuteScalarAsync(execAsync: false, CancellationToken.None).GetAwaiter().GetResult();
}


```

仔细看上面这段代码，不觉让人吸了一口凉气，所谓的同步方式竟然是用`异步方法简单包装` 而来的，这种异步混用同步的方式很容易导致线程饥饿，即线程池中已无可用线程来唤醒 GetResult() 下的 Event 事件，这个我准备后面用一篇文章详细来聊一下线程饥饿，这里用`C#内功修炼训练营`中的一张图来演示下.NET8 中异步在线程池中的走法。


![](https://img2024.cnblogs.com/blog/214741/202412/214741-20241220122814063-1263517898.png)


### 2\. 线程饥饿的现场


问题方法给大家列出来的，接下来用 windbg 看下dump中的故障现场吧。


1. 某考试系统的故障


看故障现象比较简单，使用 `!tp` 和 `!tpq` 即可，输出如下：



```

0:000> !tp
Using the Portable thread pool.

CPU utilization:  1%
Workers Total:    268
Workers Running:  268
Workers Idle:     0
Worker Min Limit: 4
Worker Max Limit: 32767

0:000> !sos tpq
global work item queue________________________________
0x000002410E750218 Microsoft.AspNetCore.Server.IIS.Core.IISHttpContextOfT
0x000002410E7505A0 Microsoft.AspNetCore.Server.IIS.Core.IISHttpContextOfT
0x000002410E750928 Microsoft.AspNetCore.Server.IIS.Core.IISHttpContextOfT
...
local per thread work items_____________________________________
0x0000024114903310 System.Runtime.CompilerServices.AsyncTaskMethodBuilder+AsyncStateMachineBoxd__23>


```

![](https://img2024.cnblogs.com/blog/214741/202412/214741-20241220122814051-131311025.png)


从卦中可以看到线程池中目前有268个线程，此时都处于运行状态，并且线程池的全局队列积压了`1000+`的任务没有处理，接下来使用 `~*e !clrstack` 观察每个线程都在做什么。



```

0:287> !clrstack
OS Thread Id: 0x39ec (287)
        Child SP               IP Call Site
000000858C5FD1B8 00007ffc95ca04e4 [HelperMethodFrame_1OBJ: 000000858c5fd1b8] System.Threading.Monitor.ObjWait(Int32, System.Object)
000000858C5FD2E0 00007ffc087cccc9 System.Threading.Monitor.Wait(System.Object, Int32) [/_/src/coreclr/System.Private.CoreLib/src/System/Threading/Monitor.CoreCLR.cs @ 156]
000000858C5FD310 00007ffc087cd027 System.Threading.ManualResetEventSlim.Wait(Int32, System.Threading.CancellationToken) [/_/src/libraries/System.Private.CoreLib/src/System/Threading/ManualResetEventSlim.cs @ 561]
000000858C5FD3D0 00007ffc087cc4f2 System.Threading.Tasks.Task.SpinThenBlockingWait(Int32, System.Threading.CancellationToken) [/_/src/libraries/System.Private.CoreLib/src/System/Threading/Tasks/Task.cs @ 3072]
000000858C5FD440 00007ffc087cc099 System.Threading.Tasks.Task.InternalWaitCore(Int32, System.Threading.CancellationToken) [/_/src/libraries/System.Private.CoreLib/src/System/Threading/Tasks/Task.cs @ 3007]
000000858C5FD4C0 00007ffc08796cc6 System.Runtime.CompilerServices.TaskAwaiter.HandleNonSuccessAndDebuggerNotification(System.Threading.Tasks.Task, System.Threading.Tasks.ConfigureAwaitOptions) [/_/src/libraries/System.Private.CoreLib/src/System/Runtime/CompilerServices/TaskAwaiter.cs @ 111]
000000858C5FD500 00007ffc086ffbc4 xxxx.UpdateAnswerUrl(System.String, Int32, System.Collections.Generic.Dictionary`2)


```

发现这些线程都卡在 `xxxx.UpdateAnswerUrl` 方法上，那到底卡在方法的何处呢？可以用 `!U /d 00007ffc086ffbc4` 观察方法的反汇编代码，看看这个00007ffc086ffbc4停留在何处？输出如下：



```
0:000> !U /d 00007ffc086ffbc4
Normal JIT generated code
xxx.UpdateAnswerUrl(System.String, Int32, System.Collections.Generic.Dictionary`2)
...
00007ffc`086ffb79 ff15114bb9fe    call    qword ptr [00007ffc`07294690] (System.Runtime.CompilerServices.AsyncMethodBuilderCore.Start[[MySql.Data.MySqlClient.MySqlCommand+d__117, MySql.Data]](d__117 ByRef), mdToken: 000000000600646B)
00007ffc`086ffb7f 488b8c2468010000 mov     rcx,qword ptr [rsp+168h]
00007ffc`086ffb87 4885c9          test    rcx,rcx
00007ffc`086ffb8a 0f84890c0000    je      00007ffc`08700819
00007ffc`086ffb90 3809            cmp     byte ptr [rcx],cl
00007ffc`086ffb92 48898c2498010000 mov     qword ptr [rsp+198h],rcx
00007ffc`086ffb9a 488d8c2498010000 lea     rcx,[rsp+198h]
00007ffc`086ffba2 48baf02b5006fc7f0000 mov rdx,7FFC06502BF0h (MT: System.Runtime.CompilerServices.TaskAwaiter`1[[System.Object, System.Private.CoreLib]])
00007ffc`086ffbac ff158e7cdefd    call    qword ptr [00007ffc`064e7840] (System.Runtime.CompilerServices.TaskAwaiter`1[[System.__Canon, System.Private.CoreLib]].GetResult(), mdToken: 00000000060065F0)
00007ffc`086ffbb2 48898424e8000000 mov     qword ptr [rsp+0E8h],rax
00007ffc`086ffbba eb0d            jmp     00007ffc`086ffbc9
00007ffc`086ffbbc 33d2            xor     edx,edx
00007ffc`086ffbbe ff1544d4bffd    call    qword ptr [00007ffc`062fd008] (System.Runtime.CompilerServices.TaskAwaiter.HandleNonSuccessAndDebuggerNotification(System.Threading.Tasks.Task, System.Threading.Tasks.ConfigureAwaitOptions), mdToken: 00000000060065E4)
>>> 00007ffc`086ffbc4 e960ffffff      jmp     00007ffc`086ffb29


```

从汇编代码中可以观测它是在获取 `ExecuteScalarAsync` 方法的 Result 结果，有了这个信息就可以翻源代码了，截图如下：


![](https://img2024.cnblogs.com/blog/214741/202412/214741-20241220122814038-228777954.png)


最终就发现了ExecuteScalar下面的荒唐一幕。。。


2. 某跟踪埋点系统的故障


埋点系统也是一样的问题，使用 `!tp` 观察到线程池有 602 个线程都处于运行状态，输出如下：



```

0:000> !tp
Using the Portable thread pool.

CPU utilization:  11%
Workers Total:    602
Workers Running:  602
Workers Idle:     0
Worker Min Limit: 32
Worker Max Limit: 32767


```

然后通过 `~*e !clrstack` 观察发现线程都处于 `Open()` 方法中，输出如下：



```

OS Thread Id: 0x1a9d4 (23)
        Child SP               IP Call Site
0000007AD4DBE228 00007ff9feb70b24 [HelperMethodFrame_1OBJ: 0000007ad4dbe228] System.Threading.Monitor.ObjWait(Int32, System.Object)
0000007AD4DBE350 00007ff9b655d55e System.Threading.Monitor.Wait(System.Object, Int32) [/_/src/coreclr/System.Private.CoreLib/src/System/Threading/Monitor.CoreCLR.cs @ 156]
0000007AD4DBE380 00007ff9b656860e System.Threading.ManualResetEventSlim.Wait(Int32, System.Threading.CancellationToken) [/_/src/libraries/System.Private.CoreLib/src/System/Threading/ManualResetEventSlim.cs @ 561]
0000007AD4DBE420 00007ff9b6581729 System.Threading.Tasks.Task.SpinThenBlockingWait(Int32, System.Threading.CancellationToken) [/_/src/libraries/System.Private.CoreLib/src/System/Threading/Tasks/Task.cs @ 3072]
0000007AD4DBE4A0 00007ff9b6581516 System.Threading.Tasks.Task.InternalWaitCore(Int32, System.Threading.CancellationToken) [/_/src/libraries/System.Private.CoreLib/src/System/Threading/Tasks/Task.cs @ 3007]
0000007AD4DBE520 00007ff959e9e9f4 System.Runtime.CompilerServices.TaskAwaiter.HandleNonSuccessAndDebuggerNotification(System.Threading.Tasks.Task, System.Threading.Tasks.ConfigureAwaitOptions) [/_/src/libraries/System.Private.CoreLib/src/System/Runtime/CompilerServices/TaskAwaiter.cs @ 111]
0000007AD4DBE560 00007ff95752e95b MySql.Data.MySqlClient.MySqlConnection.Open()
...


```

可恶的是 Open() 方法内部也是用 `异步转同步` 实现的，真的无语了。


### 3\. 解决方法


要想解决这个问题，大概两种方法吧。


1. 使用纯异步写法，这也是高版本 MySql.Data 极力推荐的，不然就给你埋坑。。。
2. 退回到低版本的 MySql.Data，继续使用真正的同步版写法。


## 三：总结


挺意外的是 MySql.Data 项目在 github：[https://github.com/mysql/mysql\-connector\-net](https://github.com) 上没开 issue 栏。


![](https://img2024.cnblogs.com/blog/214741/202412/214741-20241220122814046-969721033.png)


这就无法让社区开发者介入，真的很奇葩，只能在这里给大家做个预警吧。
![图片名称](https://images.cnblogs.com/cnblogs_com/huangxincheng/345039/o_210929020104%E6%9C%80%E6%96%B0%E6%B6%88%E6%81%AF%E4%BC%98%E6%83%A0%E4%BF%83%E9%94%80%E5%85%AC%E4%BC%97%E5%8F%B7%E5%85%B3%E6%B3%A8%E4%BA%8C%E7%BB%B4%E7%A0%81.jpg)


 本博客参考[飞数机场](https://ze16.com)。转载请注明出处！
