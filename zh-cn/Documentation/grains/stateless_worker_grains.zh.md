---
layout: page
title: Stateless Worker Grains
---

# 无状态工人Grains

默认情况下，Orleans运行时在集群中创建的grain不超过一次激活。这是虚拟角色模型最直观的表达方式，每个粒度对应一个具有唯一类型/标识的实体。但是，也有一些情况下，应用程序需要执行与系统中特定实体无关的功能性无状态操作。例如，如果客户端发送的请求带有压缩的有效负载，而这些负载需要在将它们路由到目标粒度进行处理之前进行解压缩，那么这种解压缩/路由逻辑就不会绑定到应用程序中的特定实体，并且可以很容易地进行扩展。

当`[无状态 Worker]`属性应用于grain类，它向Orleans运行时指示该类的grains应被视为**无状态 Worker**Grains。**无状态 Worker**grains具有以下特性，使得它们的执行与普通grains类的执行非常不同。

1.  Orleans运行时可以并且将在集群的不同silos上创建多个无状态工作线程的激活。
2.  对无状态工作粒度的请求总是在本地执行，即在发出请求的同一个silos上执行，或者由silos上运行的粒度发出，或者由silos的客户端网关接收。因此，从其他粒度或客户端网关调用无状态工作线程不会引发远程消息。
3.  如果已经存在的工作线程繁忙，Orleans运行时会自动创建无状态工作线程的额外激活。运行时为每个silos创建的无状态工作线程的最大激活数默认由计算机上的CPU内核数限制，除非可选的`maxLocalWorkers公司`争论。
4.  由于2和3，无状态的Worker-grain激活不能单独寻址。对无状态工作线程粒度的两个后续请求可以通过对其进行不同的激活来处理。

无状态工作线程提供了一种直接的方法来创建一个自动管理的grain激活池，该池根据实际负载自动伸缩。运行时总是以相同的顺序扫描可用的无状态工作线程粒度激活。因此，它总是将请求发送到它可以找到的第一个空闲本地激活，并且只有在所有以前的激活都很忙的情况下才能到达最后一个。如果所有的激活都很忙并且还没有达到激活限制，它会在列表的末尾再创建一个激活，并将请求发送给它。这意味着，当对无状态工作线程的请求速率增加，并且现有的激活当前都很忙时，运行时会将其激活池扩展到最大限度。相反，当负载下降时，并且可以通过较少数量的无状态工作线程的激活来处理，则列表末尾的激活将不会得到分派给它们的请求。它们将变为空闲，并最终被标准激活收集过程停用。因此，激活池最终将缩小以匹配负载。

下面的示例定义了一个无状态的Worker grain类`自述剑尾草`使用默认的最大激活数限制。

```csharp
[StatelessWorker]
public class MyStatelessWorkerGrain : Grain, IMyStatelessWorkerGrain
{
 ...
}
```

对无状态Worker-grain的调用与对任何其他grain的调用相同。唯一的区别是，在大多数情况下，使用单个grainsID，0或`Guid。空`. 当需要多个无状态工作线程粒度池时，可以使用多个grain ID，每个ID一个。

```csharp
var worker = GrainFactory.GetGrain<IMyStatelessWorkerGrain>(0);
await worker.Process(args);
```

这一个定义了一个无状态的Worker-grain类，每个silos只有一个grain激活。

```csharp
[StatelessWorker(1)] // max 1 activation per silo
public class MyLonelyWorkerGrain : ILonelyWorkerGrain
{
 ...
}
```

请注意`[无状态 Worker]`属性不会更改目标grain类的可重入性。与其他任何粒度一样，无状态工作粒度在默认情况下是不可重入的。通过添加一个`[可重入]`属性设置为grain类。

## 州

“无状态工作人员”的“无状态”部分并不意味着无状态工作人员不能有状态，并且仅限于执行功能性操作。与其他任何粒度一样，无状态工作线程可以加载所需的任何状态并将其保存在内存中。这只是因为一个无状态的Worker-grain的多个激活可以在集群的同一个和不同的竖井上创建，所以没有一个简单的机制来协调不同激活所保持的状态。

有几个有用的模式涉及到无状态工作者保持状态。

### 扩展热缓存项

对于具有高吞吐量的热缓存项，将每个这样的项保存在无状态工作进程中会使其a）在silos中自动横向扩展并跨群集中的所有silos；b）使数据始终在通过其客户端网关接收客户端请求的silos上本地可用，这样就可以在不需要额外的网络跃点到另一个silos的情况下响应请求。

### 减少样式聚合

在某些场景中，应用程序需要计算集群中特定类型的所有粒度的特定度量，并定期报告聚合。例如，报告每个游戏地图上的玩家数量、VoIP呼叫的平均持续时间等。如果成千上万或数百万个grains中的每一个都向单个全局聚合器报告其指标，聚合器将立即过载，无法处理大量报告。另一种方法是将此任务转换为2步（或更多）的reduce样式聚合。第一层聚合是通过报告grain来完成的，将它们的度量发送给无状态的Worker pre-aggregation grain。Orleans运行时将自动为每个silos创建多个无状态Worker grain的激活。由于所有这些调用都将在本地处理，而不进行远程调用或消息序列化，因此这种聚合的成本将大大低于远程情况下的成本。现在，每个预聚合无状态工作进程粒度激活，可以独立地或与其他本地激活协作，将它们的聚合报告发送到全局最终聚合器（或在必要时发送到另一个缩减层），而不会使其过载。