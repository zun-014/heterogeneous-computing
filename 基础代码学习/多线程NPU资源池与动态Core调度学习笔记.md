##### 代码

[rknn-multi-threaded/rknnpool.py at nosigmoid · leafqycc/rknn-multi-threaded](https://github.com/leafqycc/rknn-multi-threaded/blob/nosigmoid/rknnpool.py)

    多线程 NPU 资源池并行推理示例
    from queue import Queue
    from rknnlite.api import RKNNLite
    from concurrent.futures import ThreadPoolExecutor, as_completed
    
    
    def initRKNN(rknnModel="./rknnModel/yolov5s.rknn", id=0):
        rknn_lite = RKNNLite()
        ret = rknn_lite.load_rknn(rknnModel)
        if ret != 0:
            print("Load RKNN rknnModel failed")
            exit(ret)
        if id == 0:
            ret = rknn_lite.init_runtime(core_mask=RKNNLite.NPU_CORE_0)
        elif id == 1:
            ret = rknn_lite.init_runtime(core_mask=RKNNLite.NPU_CORE_1)
        elif id == 2:
            ret = rknn_lite.init_runtime(core_mask=RKNNLite.NPU_CORE_2)
        elif id == -1:
            ret = rknn_lite.init_runtime(core_mask=RKNNLite.NPU_CORE_0_1_2)
        else:
            ret = rknn_lite.init_runtime()
        if ret != 0:
            print("Init runtime environment failed")
            exit(ret)
        print(rknnModel, "\t\tdone")
        return rknn_lite
    
    
    def initRKNNs(rknnModel="./rknnModel/yolov5s.rknn", TPEs=1):
        rknn_list = []
        for i in range(TPEs):
            rknn_list.append(initRKNN(rknnModel, i % 3))
        return rknn_list
    
    
    class rknnPoolExecutor():
        def __init__(self, rknnModel, TPEs, func):
            self.TPEs = TPEs
            self.queue = Queue()
            self.rknnPool = initRKNNs(rknnModel, TPEs)
            self.pool = ThreadPoolExecutor(max_workers=TPEs)
            self.func = func
            self.num = 0
    
        def put(self, frame):
            self.queue.put(self.pool.submit(
                self.func, self.rknnPool[self.num % self.TPEs], frame))
            self.num += 1
    
        def get(self):
            if self.queue.empty():
                return None, False
            fut = self.queue.get()
            return fut.result(), True
    
        def release(self):
            self.pool.shutdown()
            for rknn_lite in self.rknnPool:
                rknn_lite.release()

##### 分析

```
输入数据 / 视频帧
        ↓
   Python任务提交
        ↓
 ThreadPoolExecutor
        ↓
   RKNN实例资源池
   ├─ 实例0 → NPU Core0
   ├─ 实例1 → NPU Core1
   └─ 实例2 → NPU Core2
        ↓
 rknn_lite.inference()
        ↓
   NPU并行推理
        ↓
     推理结果
```

当前实例代码仅实现了简单的轮询式的NPU并行处理同一模型，例如 Frame0 由 Core0 推理、Frame1 由 Core1 推理、Frame2 由 Core2 推理，从而提高 RK3588 三个 NPU Core 的整体利用率和系统吞吐量。因此它**不会判断**某个 Core 当前是否繁忙、是否存在任务积压、温度是否过高或者任务是否具有更高优先级。

###### 动态调度（目的）：

对于当前的这个基础项目而言，RKNN 提供“选择并调用某个 NPU Core”的执行接口，如果在**上层**增加**实时负载、队列、温度、Deadline、健康状态等**感知信息后，由调度器先做决策，再调用对应的 RKNN API，就可以形成动态 Core 调度。

例如采集各 NPU Core 的任务队列、推理时间、利用率、温度、健康状态，以及任务本身的 Deadline 和优先级，再由上层 Scheduler 根据这些信息动态选择最合适的 RKNNLite 实例执行任务。

```
负载感知

+ Deadline
+ 队列状态
+ 健康状态
......
      ↓
  Scheduler
      ↓
  选择 RKNN实例0/1/2
      ↓
  对应 NPU Core0/1/2
```

