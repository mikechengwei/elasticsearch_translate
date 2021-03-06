# 线程池

每个节点都有一些线程池来优化线程内存的消耗，按节点来配置管理。有些线程池还拥有与之关联的队列配置，用来允许挂住一些未处理的请求，而不是丢弃它。

下面仅列出来了一些重要的线程池：

generic

&emsp;&emsp;*用于通用的操作（例如：后台节点发现），线程池类型为 **scaling**。*

index

&emsp;&emsp;*用于index/delete操作，线程池类型为 **fixed**， 大小的为`处理器数量`，队列大小为`200`，最大线程数为 `1 + 处理器数量`。*

search

&emsp;&emsp;*用于count/search/suggest操作。线程池类型为 **fixed**， 大小的为`int((处理器数量 * 3) / 2) +1`，队列大小为`1000`。*

get

&emsp;&emsp;*用于get操作。线程池类型为 **fixed**，大小的为`处理器数量`，队列大小为`1000`。*

bulk

&emsp;&emsp;*用于bulk操作，线程池类型为 **fixed**， 大小的为`处理器数量`，队列大小为`200`，该池的最大线程数为 `1 + 处理器数量`。*

percolate

&emsp;&emsp;*用于percolate操作，线程池类型为 **fixed**， 大小的为`处理器数量`，队列大小为`1000`*

snapshot

&emsp;&emsp;*用于snaphost/restore操作。线程池类型为 **scaling**，线程保持存活时间为5分钟，最大线程数为`min(5, (处理器数量)/2)`。*

warmer

&emsp;&emsp;*用于segment warm-up操作。线程池类型为 **scaling**，线程保持存活时间为5分钟，最大线程数为`min(5, (处理器数量)/2)`。*

refresh

&emsp;&emsp;*用于refresh操作。线程池类型为 **scaling**，线程空闲保持存活时间为5分钟，最大线程数为`min(10, (处理器数量)/2)`。*

listener

&emsp;&emsp;*主要用于Java客户端线程监听器被设置为true时执行动作。线程池类型为 **scaling**，最大线程数为`min(10, (处理器数量)/2)`。*

**更改指定线程池可以通过设置指定类型的参数来实现; 例如，改变`index`线程池有更多的线程：**

```yml
thread_pool:
    index:
        size: 30
```

## 线程池类型

以下是线程池的类型和各自的参数：

### fixed（固定）

`fixed`线程池拥有固定数量的线程来处理请求，在没有空闲线程时请求将被挂在队列中（可选配）。

`size`参数用来控制线程的数目，默认为数量为5。

`queue_size`参数可以控制在没有空闲线程时，能排队挂起的请求数。默认情况下它被设置为`-1`，这意味着它是无限的。当一个请求进来时如果队列已满，请求将被中止。

```yml
thread_pool:
    index:
        size: 30
        queue_size: 1000
```

### scaling（弹性）

`scaling`线程池拥有的线程数量是动态的。这个数字介于`core`和`max`参数的配置之间变化。

`keep_alive`参数用来控制线程在线程池中空闲的最长时间。（译者注：线程池中线程的空闲时间超过此值、且池中的线程数量不少于`core`时，线程会被销毁）。

```yml
thread_pool:
    warmer:
        core: 1
        max: 8
        keep_alive: 2m
```

## 处理器设置

Elasticsearch会自动探测处理器的数量，并且线程池的设置将基于它自动设置。在某些情况下，你可能需要自己覆盖自动探测的处理器数量，这可以通过显式设置`processors`参数来进行设置。

```yml
processors: 2
```

下面有几个场景是需要明确的覆盖的`processors`设置：

1. 如果要在同一主机上运行Elasticsearch的多个实例，但希望Elasticsearch线程池的大小只根据一部分CPU来设置，这时你应该通过`processors`参数来重设处理器数量。（例如，如果你在16核的机器上运行两个Elasticsearch实例，可以设置`processors`为`8`）。请注意，这是一个专家级的场景，这种情况不仅仅是设置一下`processors`就行的，因为还有更多复杂的其他因素需要设置，譬如修改垃圾收集器线程数量、绑定进程到CPU等。
1. 自动探测处理器数量的默认上限是32。这意味着，在具有超过32个处理器的系统中，Elasticsearch的线程池大小会受限于32个处理器。加入此限制是问了避免在没有正确调整操作系统的`ulimit`最大进程数时创建了过多的线程，在你适当的调整`ulimit`情况下，则可以显式设置此`processors`参数。
1. 有时候被错误地检测出处理器的数量，在这种情况下，明确设置`processors`将解决此问题。

若要检查自动探测的处理器数量，可以使用节点信息API通过os标志来查看。