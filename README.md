
## 一：背景


### 1\. 讲故事


最近时间相对比较宽裕，多写点文章来充实社区吧，这篇文章主要还是来自于最近遇到的几例线程饥饿`(Task.Result)`引发的一系列的反思和总结，我觉得.NET8容易引发饥饿的原因，更多的在于异步回调之后底层会反复的将结果丢到线程池所致，因为数据进线程池容易，再用线程到池中去捞就没有那么简单了，可能今天的话题比较有争议，当然我个人的思考也不见得一定对，算是给大家提供一个角度吧，话不多说，开干！


## 二：为什么会容易饥饿


### 1\. 测试代码


为了方便讲述异步回调的路径，这里我用简单的 `FileStream` 的异步读取来演示，当然实际的场景更多的是网络IO，最后我再上一个 .NET6 和 .NET8 的对比，先看一下参考代码。



```

    internal class Program
    {
        static void Main(string[] args)
        {
            UseAwaitAsync();

            Console.ReadLine();
        }

        static async Task<string> UseAwaitAsync()
        {
            string filePath = "D:\\dumps\\trace-1\\GenHome.DMP";
            Console.WriteLine($"{DateTime.Now.ToString("yyyy-MM-dd HH:mm:ss:fff")} 请求发起...");
            using (FileStream fileStream = new FileStream(filePath, FileMode.Open, FileAccess.Read, FileShare.Read, bufferSize: 16, useAsync: true))
            {
                byte[] buffer = new byte[fileStream.Length];

                int bytesRead = await fileStream.ReadAsync(buffer, 0, buffer.Length);

                string content = Encoding.UTF8.GetString(buffer, 0, bytesRead);

                var query = $"{DateTime.Now.ToString("yyyy-MM-dd HH:mm:ss:fff")} 获取到结果:{content.Length}";

                Console.WriteLine(query);

                return query;
            }
        }
    }


```

卦中 await 之后的回调，很多人可能会想当然的以为是`IO线程`一撸到底，其实在.NET8中并不是这样的，它会经历两次 Enqueue 到线程池，步骤如下：


1. IO线程 封送 event 到 线程池队列。
2. Worker线程 读取 event 拆解出 ValueTaskSourceAsTask(ReadAsync) 再次入 线程池队列。
3. Worker线程 读取 ValueTaskSourceAsTask(ReadAsync) 拆解出编译器生成的状态机`d__1`，回到用户代码。


这里我姑且定义成三阶段吧，可能有些朋友有点模糊，我画一张简图给大家辅助一下。


![](https://img2024.cnblogs.com/blog/214741/202501/214741-20250107144032748-177103632.png)


代码和图都有了，接下来就是眼见为实的阶段了。


### 2\. 如何眼见为实


这个相对来说比较简单，在合适的位置埋上断点，然后观察线程栈即可。


1. 观察第一阶段


自 C\# 重写了ThreadPool之后，底层会用一个单独的线程轮询IO完成端口队列（GetQueuedCompletionStatusEx），参考代码如下：



```

    internal sealed class PortableThreadPool
    {
        private unsafe void Poll()
        {
            int num;
            while (Interop.Kernel32.GetQueuedCompletionStatusEx(this._port, this._nativeEvents, 1024, out num, -1, false))
            {
                for (int i = 0; i < num; i++)
                {
                    Interop.Kernel32.OVERLAPPED_ENTRY* ptr = this._nativeEvents + i;
                    if (ptr->lpOverlapped != null)
                    {
                        this._events.BatchEnqueue(new PortableThreadPool.IOCompletionPoller.Event(ptr->lpOverlapped, ptr->dwNumberOfBytesTransferred));
                    }
                }
                this._events.CompleteBatchEnqueue();
            }
            ThrowHelper.ThrowApplicationException(Marshal.GetHRForLastWin32Error());
        }
    }


```

从卦中看，一旦 GetQueuedCompletionStatusEx 获取到了数据就开始封送 event，并投送到线程池的`高优先级队列`中,我们可以在 UnsafeQueueHighPriorityWorkItemInternal 上下断点即可，然后观察线程栈，截图如下：


![](https://img2024.cnblogs.com/blog/214741/202501/214741-20250107144032769-1646360453.png)


2. 观察第二阶段


当IO线程将数据丢到队列之后，接下来就需要用 Worker线程 去取了，这里就有了一个重大隐患，这个隐患在于如果当前存在线程饥饿，而线程的动态注入又比较慢，所以这个event存在不能及时取出来的情况。


按照模型图描述，这个阶段是从 event 中拆解出 ValueTaskSourceAsTask，这中间还涉及到了 ThreadPoolBoundHandleOverlapped 的解包逻辑，我在上篇[聊一聊 C\#异步中的Overlapped是如何寻址的](https://github.com)和大家聊过，这里就不赘述了，接下来在`ManualResetValueTaskSourceCore.SignalCompletion()` 上下一个断点观察。


![](https://img2024.cnblogs.com/blog/214741/202501/214741-20250107144032752-364226040.png)


上面卦中的 `_continuationState` 就是最终拆解的 ValueTaskSourceAsTask（ReadAsync），截图如下：


![](https://img2024.cnblogs.com/blog/214741/202501/214741-20250107144032729-1492230148.png)


有些朋友可能会有疑惑，`ReadAsync` 返回的是 `Task` ，怎么就变成了 `ValueTaskSourceAsTask` 呢？这是因为 ReadAsync 的底层做了一个 `ValueTask -> Task` 的转换，参考代码如下：



```

        public override Task<int> ReadAsync(byte[] buffer, int offset, int count, CancellationToken cancellationToken)
        {
            ValueTask<int> valueTask = this.ReadAsync(new Memory(buffer, offset, count), cancellationToken);
            if (!valueTask.IsCompletedSuccessfully)
            {
                return valueTask.AsTask();
            }
            return this._lastSyncCompletedReadTask.GetTask(valueTask.Result);
        }


```

反正不管怎么说，确实是真真切切的再次将数据(ValueTaskSourceAsTask) 丢入了线程池的线程本地队列，可二次丢入又放大了饥饿的风险。


3. 观察第三阶段


数据进了队列之后，需要线程池线程再次提取，这个逻辑就比较简单了，提取 `ValueTaskSourceAsTask` 中的延续字段 `continuationObject` 来解构状态机最终回到用户代码，要想观察直接在用户方法 UseAwaitAsync() 的 await 之后下一个断点即可。


![](https://img2024.cnblogs.com/blog/214741/202501/214741-20250107144032746-278512009.png)


### 3\. .NET6 会这样吗


很多朋友可能会说 .NET8 是这样，那之前的版本也是这样吗？ 也有一些朋友可能会说，我的饥饿发生在 `网络IO`，并没有看到类似 `文件IO` 的情况。


在我的dump分析之旅中，确实几乎所有的饥饿都发生在 `网络IO`上，并且 .NET6 和 .NET8 在 `网络IO` 上的行为已经完全不一样了。


1. .NET6 是IO线程 一撸到底。
2. .NET8 则需要 Worker线程 做二次处理。


说了这么多，我们上一个`网络IO`的例子，然后观察 .NET6 和 .NET8 在处理回调上的不同，参考代码如下：



```

    internal class Program
    {
        static async Task Main(string[] args)
        {
            var task = await GetContentLengthAsync("http://baidu.com");

            Console.ReadLine();
        }
        static async Task<int> GetContentLengthAsync(string url)
        {
            using (HttpClient client = new HttpClient())
            {
                var content = await client.GetStringAsync(url);

                Debug.WriteLine($"线程编号:{Environment.CurrentManagedThreadId}, content.length={content.Length}");

                Debugger.Break();

                return content.Length;
            }
        }
    }


```

1. .NET6 下的WinDbg观察


![](https://img2024.cnblogs.com/blog/214741/202501/214741-20250107144032732-877209947.png)


从卦中可以看到 `tid=10` 后面有一个 `Threadpool Completion Port`标记，这就表明确实是 IO线程 一撸到底。


2. .NET8 下的WinDbg观察


![](https://img2024.cnblogs.com/blog/214741/202501/214741-20250107144032746-1061913083.png)


从卦中可以看到 `tid=9` 后面是 `Threadpool Worker`标记，这就说明复杂了哈。。。


## 三：总结


可以肯定的是减少callback重入队列次数可以尽可能的避免线程饥饿，但怎么说呢？`.NET8`的线程池综合性能绝对比 `.NET6` 要强悍的多，但.NET8中的设计理念可能也不能达到100%的全域领跑，可能在某些1%的场景下还不如 .NET6 的简单粗暴。
![图片名称](https://images.cnblogs.com/cnblogs_com/huangxincheng/345039/o_210929020104%E6%9C%80%E6%96%B0%E6%B6%88%E6%81%AF%E4%BC%98%E6%83%A0%E4%BF%83%E9%94%80%E5%85%AC%E4%BC%97%E5%8F%B7%E5%85%B3%E6%B3%A8%E4%BA%8C%E7%BB%B4%E7%A0%81.jpg)


 本博客参考[FlowerCloud机场](https://hanlianfangzhi.com)。转载请注明出处！
