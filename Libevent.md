# Libevent

**Libevent** 是一个用于编写快速可移植非阻塞 IO 的库，其设计理念为：

1. 可移植
 使用 Libevent 编写的程序应当能在所有 Libevent 支持的平台上运行。即使在某些情况下并
没有很好的方法完成非阻塞 IO，Libevent 也应当可以马马虎虎地支持，以保证您的程序能在受
限环境中运行。
2. 高效
 Libevent 尽可能使用各个平台上可用且最快的非阻塞 IO 实现，并且不会引入太多开销。
3. 可扩展
 Libevent 被设计成在程序需要数万个活跃套接字时仍然可用。
4. 便捷
 不管在任何时候，使用 Libevent 编写程序最自然的方式应当是稳定的、可移植的。

Libevent 被划分成以下几个组件：

**evutil**

 通用功能，抽象出不同平台的网络实现之间的差异。

**event 和 event_base**  

event 和 event_base 是 Libevent 的核心。它提供了一个抽象的 API，用于与各种平台特
定的、基于事件的非阻塞 IO 后端进行交互。它可以让你知道套接字什么时候是可读或可写的，
执行基本的超时功能，以及检测操作系统信号。

**bufferevent**

 bufferevent 为 Libevent 的基于事件的内核提供了更方便的包装。它可以让您在 IO 实际发
生的时候进行缓冲区的读写操作，而不是在套接字就绪时通知您。
 bufferevent 接口支持多种后端，所以它可以利用系统提供提供的更快的方式来进行非阻塞
IO，例如 Windows 的 IOCP API。

**evbuffer**

 这个模块实现了 bufferevent 底层的缓冲区，并提供了高效且/或方便的函数。

**evhttp**

 一个简单的 HTTP 客户端/服务器实现。

**evdns**

 一个简单的 DNS 客户端/服务器实现。

**evrpc**
 一个简单的 RPC 实现。


# R1：设置 Libevent
 Libevent 中有一些在整个进程中共享的全局设置，这些设置会影响整个库。
在调用 Libevent 库的其他任何部分之前，您必须对这些设置进行一些处理，否则 Libevent 可
能会处于不一致的状态。

## Libevent 日志消息 
Libevent 可以记录内部错误和警告。如果在编译时开启日志支持，也可以记录调试信息。默认情
况下这些消息会被写到标准错误（stderr ），您可以提供自己的日志函数以覆盖默认行为。

接口
```c++
 #define EVENT_LOG_DEBUG 0
 #define EVENT_LOG_MSG   1
 #define EVENT_LOG_WARN  2
 #define EVENT_LOG_ERR   3

 typedef void (*event_log_cb)(int severity, const char *msg);
 void event_set_log_callback(event_log_cb cb);
 ```
 
要覆盖 Libevent 的日志行为，只需编写符合 event_log_cb 签名的函数，将其作为参数传递给 event_set_log_calback() 。Libevent 需要记录消息时，将会传递给您提供的函数。当需要恢复默认日志行为时，调用 event_set_log_callback() 并传递参数 NULL 即可。

通常情况下，调试日志不会开启，也不会被发送到日志回调函数。如果 Libevent 被构建成支持这些功能的话，可以手动开启。

接口
```c++
#define EVENT_DBG_NONE 0
#define EVENT_DBG_ALL 0xffffffffu
void event_enable_debug_logging(ev_uint32_t which);
```
调试日志是冗长的，并且在大多数情况下都不是必要的。调用 event_enable_debug_logging 并传递参数 EVENT_DBG_NONE 可以得到默认行为；传递 EVENT_DBG_ALL 将开启所有调试日志。

## 处理致命错误
当 Libevent 检测到一个不可恢复的内部错误（例如损坏的数据结构），其默认行为是调用 exit() 或 abort() 以退出当前进程。这些错误通常意味着某处有 bug：要么在您的代码中，要么在 Libevent 中。如果希望应用程序更优雅地处理致命错误，可以通过提供退出时需要调用的函数来重写 Libevent 的默认行为

接口
```c++
typedef void (*event_fatal_cb)(int err);
void event_set_fatal_callback(event_fatal_cb cb);
```

为了使用这些功能，首先需要为 Libevent 定义一个在遭遇致命错误时应当调用的函数并传递给 
event_set_fatal_callback() 。随后 Libevent 会在遭遇致命错误时调用您提供的函数。

## 释放 Libevent 的全局结构体 
即使已经释放了所有通过 Libevent 分配的对象，仍然会存在一些全局分配的结构体。这通常不是什么问题：一旦进程退出，这些结构体将会被销毁。但是保留这些结构体可能会使一些调试工具误认为 Libevent 出现了资源泄露。如果需要确保 Libevent 释放所有内部全局数据结构，可以调用：

接口
```c++
void libevent_global_shutdown(void);
```
该函数不会释放任何通过 Libevent 函数返回的结构体。如果希望退出之前释放所有资源，需要手动释放 event 、event_base 、bufferevent 等资源。调用 libevent_global_shutdown() 会让其他 Libevent 函数的行为无法预测，除非将其作为程序最后调用的 Libevent 函数，否则不要调用它。

# R2：创建 event_base
在使用任何有趣的 Libevent 函数之前，您需要申请一个或多个 event_base 结构体。每个结构体持有一个事件集，并可以轮询以确定哪些事件处于活跃状态。

如果 event_base 设置成使用锁，则可以在多个线程中安全地访问。然而，其事件循环只能运行在一个线程中。如果希望使用多个线程轮询 IO，则需要为每个线程都分配 event_base 。

每个 event_base 都有一个 "方法"，或者后端来检测哪些事件已经就绪，可识别的方法有：
- select
- poll
- epoll
- kqueue
- evport
- win32

用户可以通过环境变量禁用特定后端。例如，可以通过设置 EVENT_NOKQUEUE 环境变量来禁用 kqueue ，其他后端同理。如果想要在程序内部禁用某个后端，使用event_config_avoid_method()

## 创建默认的 event_base
event_base_new() 函数申请并返回一个新的包含默认配置 并返回一个指向 event_base 。该函数检查环境变量event_base 的指针，如果发生错误则返回 NULL 。

在选择方法时，该函数会选择受操作系统支持的最快的方法。

接口
```c++
struct event_base *event_base_new(void);
```
event_base_new() 函数声明在 <event2/event.h>

## 创建复杂的 event_base
如果想对 
event_base 的种类获取更多控制权，您需要使用 event_config 。event_config 是一个不透明的结构，保存了有关 event_base 的首选项的信息。需要创建 event_base 时，将 event_config 传递给 event_base_new_with_config() 函数。

接口
```c++
struct event_config *event_config_new(void);
struct event_base *event_base_new_with_config(const struct event_config *cfg);
void event_config_free(struct event_config *cfg);
```

## 释放 event_base
当 event_base 使用完毕，您可以通过 event_base_free() 释放。

接口
```c++
void event_base_free(struct event_base *base);
```

注意：这个函数不会释放当前与 event_base 关联的事件，或者关闭它们的套接字，或者释放任何它们的指针。

## 设置 event_base 优先级
Libevent 支持为同一个事件设置多个优先级。然后，默认情况下 event_base 只支持一个优先级。您可以通过调用event_base_priority_init() 来设置  event_base 的优先级。

接口
```c++
int event_base_priority_init(struct event_base *base, int n_priorities);
```
该函数成功返回 0，失败返回 -1。参数 base 是需要修改的 event_base ，n_priorities 是支持的优先级的数量，该值至少是 1。事件的优先级范围为 0（最重要） 到 n_priorities  - 1（最不重要）。

注意：您必须在任何事件激活之前调用该函数，最好在创建 event_base 后立刻调用。

event_base_getnpriorities() 获取某个 event_base 当前支持的优先级数量。

接口
```c++
int event_base_get_npriorities(struct event_base *base);
```

返回值是 event_base 中设置的支持的优先级数量。如果 event_base_get_npriorities() 返回 3，则允许的优先级值为 0，1 和 2。

## fork() 后重新初始化 event_base 
并非所有事件后端都在调用 fork() 后仍然可用。因此，如果您的程序在创建新进程时使用了 fork() 或相关的系统调用，并且希望 fork 之后继续使用 event_base ，就需要重新初始化 

接口
```c++
int event_reinit(struct event_base *base);
```
该函数成功返回 0，失败返回 -1。

## 废弃的 event_base 函数
老版本的 Libevent 非常依赖于 "当前" event_base 的概念，"当前" 线程共享的全局设置。如果您忘记指定 event_base 是一个所有event_base ，则会使用 "当前" 的。由于 event_base 不是线程安全的，这很容易出错。

老版本的 Libevent 没有 event_base_new() ，而是：

接口
```c++
struct event_base *event_init(void);
```

这个函数的行为与 event_base_new() 类似，并且会把申请的 event_base 设置成 "当前" 的。 没有其他方法可以改变 "当前" event_base 

# R3：使用事件循环
## 启动循环
一旦 event_base 注册了一些事件，您将希望 Libevent 等待事件并在触发时通知您

接口
```c++
#define EVLOOP_ONCE 0x01            
#define EVLOOP_NONBLOCK 0x02  //循环不会等待事件触发：它只会立刻检查是否有事件可以触发，如果有，则执行回调。    
// EVLOOP_NO_EXIT_ON_EMPTY 从 Libevent 2.1 开始存在
#define EVLOOP_NO_EXIT_ON_EMPTY 0x04
int event_base_loop(struct event_base *base, int flags);
```
默认情况下，
event_base_loop() 函数会一直运行直到内部没有注册事件。运行循环时，它会反复检查是否触发了任何已注册的事件（例如，读事件的文件描述符是可读的，或者到达超时事件的超时时间）。一旦有事件触发，该函数将所有触发事件标记为 "活跃的（active）" 并开始执行这些事件。

循环结束后，如果正常退出，则返回 0，若因后端出现了未处理的错误而退出则返回 -1，若因没
有任何未决或活跃事件而退出则返回 1。

方便起见，也可以调用：

接口
```c++
int event_base_dispatch(struct event_base *base);
```
event_base_dispatch() 的调用与 event_base_loop() 一样，只是无需设置标志。因此，该函数会一直运行到没有注册事件或 event_base_loopbreak() 或 event_base_loopexit() 被调用。

## 终止循环
如果想在所有事件移除之前终止循环，您可以调用两个稍有不同的函数。

接口
```c++
int event_base_loopexit(struct event_base *base,const struct timeval *tv);
int event_base_loopbreak(struct event_base *base);
```

event_base_loopexit() 函数通知 event_base 在给定的时间流逝后停止循环。如果 tv 参数是 NULL ，则立即终止。如果当前 event_base 正在执行活跃事件回调，则会继续执行，直到执行完所有回调后才退出。

event_base_loopbreak() 函数通知 event_base 立即退出循环。与event_base_loopexit(base, NULL) 不同的是，如果 event_base 正在执行活跃事件的回调，则会在完成该事件后立即退出。

同样需要注意 event_base_loopexit(base, NULL) 和 event_base_loopbreak(base) 在事件循环没有运行时的行为的不同： loopexit 安排下一次事件循环在下一轮回调完成后立即停止（就像设置了 EVLOOP_ONCE 一样），而 loopbreak 仅仅终止当前运行的循环，循环不在运行则没有任何效果。

这两个函数都在成功时返回 0，失败时返回 -1。

有时可能需要判断循环是正常退出的，还是因为 event_base_loopexit() 或event_base_loopbreak() 退出的，可以调用以下函数来判断 loopexit 或 loopbreak 是否被调用：

接口
```c++
int event_base_got_exit(struct event_base *base);
int event_base_got_break(struct event_base *base);
```
如果循环是因为调用 event_base_loopexit() 或 event_base_loopbreak() 退出的，这两个函数会分别返回真，否则返回假。下次启动事件循环会重置。

## 重新检查事件
通常，Libevent 会检查所有事件，随后运行所有最高优先级活跃事件，然后重新检查事件，依次循环。然而有时候可能需要 Libevent 在执行完当前回调后立即停止（调用其他相同优先级活跃事件的回调函数），并通知 Libevent 重新扫描。与 event_base_loopbreak() 类似，您可以通过 event_base_loopcontinue() 来完成这件事。

接口
```c++
int event_base_loopcontinue(struct event_base *);
```

如果当前没有正在执行的事件回调，则调用此函数不会产生任何影响。

# R4：处理事件
Libevent 的基本处理单元是事件。每个事件代表一个条件集合，包括：
- 文件描述符就绪，可读或可写。
- 文件描述符变成就绪状态，可读或可写（仅限于边缘触发 IO）。
- 即将超时。
- 信号发生。
- 用户触发事件。

事件拥有相似的生命周期。一旦调用 Libevent 函数创建一个事件并关联到 event_base ，事件进入已初始化状态。此时可以添加事件，使其在 event_base 中成为未决状态（事件需要先关联再添加）。当事件处于未决状态，如果可以触发事件的条件满足（例如，文件描述符改变状态或超时），该事件变为活跃状态。如果事件配置为持久（persistent），则转成未决状态；如果不是持久的，则回调时不会变成未决状态。可以通过删除将未决事件转变为非未决事件，也可以通过添加将非未决事件转变为未决。

## 构造事件对象
使用 event_new() 接口创建对象。

接口
```c++
#define EV_TIMEOUT 0x01
#define EV_READ 0x02
#define EV_WRITE 0x04
#define EV_SIGNAL 0x08
#define EV_PERSIST 0x10
#define EV_ET 0x20
typedef void (*event_callback_fn)(evutil_socket_t, short, void *);
struct event *event_new(struct event_base *base, evutil_socket_t fd,
    short what, event_callback_fn cb,void *arg);
void event_free(struct event *event);
```

event_new() 尝试申请并构造一个用于 base 的新事件。 what 参数是上述列表中标志的集合。（标志的语义将会在下方描述。）如果 fd 非负，我们将观察该文件的读或写。当事件被激活，Libevent 将会回调提供的 cb 函数，并传递参数：文件描述符 fd ，表示所有被触发事件（这里的事件是使事件对象活跃的行为或信号）的比特字段，以及在构造函数时传递的 arg 。

发生内部错误或参数不正确时， event_new() 会返回 NULL 。

新事件的状态是已初始化而非未决。调用 event_add() 可使其转变为未决状态

调用 event_free() 可以释放事件。对未决或活跃事件调用该函数是安全的：该函数会在释放前将事件状态转变为非未决和非活跃。

## 事件标志
**EV_TIMEOUT**  
 此标志表明超时时间流逝后事件会变成活跃状态。

 构造事件时 EV_TIMEOUT 标志会被忽略：您可以在添加时选择是否设置超时时间。超时时间到达时此标志会设置在回调函数的 what 参数中。

**EV_READ**  
 此标志表明事件会在给定的文件描述符可读时变成活跃状态。

**EV_WRITE**
 此标志表明事件会在给定的文件描述符可写时变成活跃状态。

**EV_SIGNAL**
 用于检测信号

**EV_PERSIST**
 表明事件是持久的

**EV_ET**
 表明如果 event_base 的后端支持边缘触发，则事件应当是边缘触发的。这个标志会影响EV_READ 和 EV_WRITE 的语义。

```
边缘触发只在文件状态改变时触发，即文件从不可读变为可读。若程序只读了一部
分，文件仍然是可读的。但是由于可读与可读之间并没有发生状态转变，因此不会再次触发。
与之对应的是水平触发，若文件未读完，则会一直触发。
```

## 关于事件持久性
默认情况下，任何未决事件活跃后（由于文件描述符可写或可读，或到达超时时间），都会在回调之前变成非未决状态。因此，如果希望让事件重新进入未决状态，您可以在回调函数内部再次调用 event_add() 。

如果事件设置了 EV_PERSIST 标志，则该事件是持久的。这意味着回调时该事件会保持未决状态。如果需将在回调内部将事件设置成非未决，您可以调用 event_del() 函数。

每次回调都会重置超时。因此，如果您的事件被设置成 EV_READ|EV_PERSIST 且超时时间为5s，则该事件会在这些情况下活跃：
- 每次套接字可写。
- 上次活跃后 5s。

## 纯超时事件（定时器）
为方便使用，有一些以 evtimer_ 开头的宏可以替代 event_* 函数来申请和操作纯超时事件。这些宏除了能简化代码没有任何其他作用。

接口
```c++
#define evtimer_new(base, callback, arg) \
    event_new((base), -1, 0, (callback), (arg))
#define evtimer_add(ev, tv) \
    event_add((ev),(tv))
#define evtimer_del(ev) \
    event_del(ev)
#define evtimer_pending(ev, tv_out) \
    event_pending((ev), EV_TIMEOUT, (tv_out))
```

## 构建信号事件
Libevent 也可以监测 POSIX 式的信号。要构建一个信号处理器，使用：

接口
```c++
#define evsignal_new(base, signum, cb, arg) \
    event_new(base, signum, EV_SIGNAL|EV_PERSIST, cb, arg)
```
除提供信号编号代替文件描述符外，其他参数与 event_new 一致。

Libevent 也提供了一些方便使用的宏来处理信号事件。

接口
```c++
#define evsignal_add(ev, tv) \
    event_add((ev),(tv))
#define evsignal_del(ev) \
    event_del(ev)
#define evsignal_pending(ev, what, tv_out) \
    event_pending((ev), (what), (tv_out))
```

## 创建用户触发事件
有时，创建可以在所有高优先级事件完成后激活并执行的事件是很有用的。任何形式的清理以及垃圾回收都是此类事件。

创建用户触发事件：
```c++
struct event *user = event_new(evbase, -1, 0, user_cb, myhandle);
```
注意不需要调用 event_add() ，只需使用下面的方式激活：
```c++
event_active(user, 0, 0);
```

## 创建不在堆上分配的事件
接口
```c++
int event_assign(struct event *event, struct event_base *base,
    evutil_socket_t fd, short what,
void (*callback)(evutil_socket_t, short, void *), void *arg);
```
除 event 参数必须指向一个未初始化的事件外， event_assign() 参数与 event_new() 参数一致。该函数成功返回 0，失败或参数无效返回 -1。

您也可以用 event_assign() 来初始化栈分配或静态分配事件。

警告  
绝对不要对 event_base 中的未决事件调用 event_assign() ，这样做会导致极难定位的错误。
如果事件已经初始化并处于未决状态，再次调用 event_assign() 之前请先调用 event_del()。

## 使事件未决和非未决
创建事件后，通过添加使其变为未决状态前，事件不会有任何行为。添加事件可以使用  
event_add ：

接口
```c++
int event_add(struct event *ev, const struct timeval *tv);
```
对非未决事件调用 event_add 会使其在配置的 event_base 中转为未决，该函数成功返回 0，失败返回 -1。如果 tv 是 NULL ，则不会设置超时时间，否则 tv 就是以秒和微秒为单位的超时时间。

如果对已经处于未决状态的事件调用 event_add() ，则会继续保持在未决状态，并且会在给定的超时时间后重新调度。如果超时时间为 NULL ，则 event_add() 没有任何效果。

接口
```c++
int event_del(struct event *ev);
```
对初始化过的事件调用 event_del 会使其非未决和非活跃。如果事件并不处在未决或活跃状态，则没有任何效果。该函数成功返回 0，失败返回 -1。

注意：如果在事件活跃但还没有机会执行回调时删除事件，则回调不会被执行。

接口
```c++
int event_remove_timer(struct event *ev);
```
可以在不删除未决事件的 IO 或信号组件的前提下移除未决时间的定时器了。如果事件没有超时组件， event_remove_timer() 没有任何作用。如果是一个纯超时事件，则event_remove_timer() 与 event_del() 效果相同。该函数成功返回 0，失败返回 -1。

## 具有优先级的事件
当多个事件同时触发，Libevent 没有定义任何关于何时回调的顺序。通过使用优先级，您可以将一些事件定义为比其他事件更重要。

每个 event_base 有与之相关的一个或多个优先级。在初始化事件后，添加到 event_base 前，可以设置事件的优先级。

接口
```c++
int event_priority_set(struct event *event, int priority);
```
优先级的取值范围是 0 到优先级数量减 1。该函数成功时返回 0，失败时返回 -1。当多个不同优先级的事件活跃时，低优先级事件不会执行。Libevent 会执行高优先级事件，然后重新检查事件。只有不再有高优先级事件活跃时，低优先级事件才会执行。

如果没有设置事件的优先级，默认是 event_base 支持的优先级数量除以 2。

## 配置单次触发事件
如果不需要多次添加事件，或者添加后立即删除，并且事件是非持久的，可以使用event_base_once() 。
接口
```c++
int event_base_once(struct event_base *, evutil_socket_t, short,
    void (*)(evutil_socket_t, short, void *), void *, const struct timeval *);
```

该函数接口与 event_new() 一致，除了不支持 EV_SIGNAL 或 EV_PERSIST 。事件会以默认优先级添加进 event_base 并执行。回调结束后，Libevent 会释放事件结构体。该函数成功返回0，失败返回 -1。

使用 event_base_once 添加的事件无法被删除或手动触发：如果希望能够取消事件，请通过常规的 event_new() 或 event_assign() 接口创建事件。

## 手动激活事件
极少数情况下，即使条件没有触发，您也希望激活某个事件。

接口
```c++
void event_active(struct event *ev, int what, short ncalls);
```
这个函数会使用 what 标志位（ EV_READ ， EV_WRITE 和 EV_TIMEOUT 的组合）激活事件 ev。该事件不需要事先处于未决状态，激活事件也不会使其未决。

# R5：辅助函数和类型
<event2/util.h> 中定义了很多函数，您可能会发现这些函数对使用 Libevent 实现可移植程序很有帮助。Libevent 内部也使用了这些类型和函数

## 基本类型
**evutil_socket_t**
除 Windows 外的绝大多数地方，套接字都是一个整型变量，操作系统按照数值次序进行分配。然而，在使用 Windows 套接字 API 时，套接字的类型是 SOCKET ，是一个类似于指针的操作系统句柄，并且接收的次序是未定义的。在 Windows 中，我们将 evutil_socket_t 定义为整型指针，能够在避免指针截断风险的同时处理 socket() 和 accept() 的输出。

定义
```c++
#ifdef WIN32
#define evutil_socket_t intptr_t
#else
#define evutil_socket_t int
#endif
```

## 定时器可移植函数
并不是所有的平台都定义了标准 timeval 操作函数，因此我们提供了自己的实现。
接口
```c++
#define evutil_timeradd(tvp, uvp, vvp) /* ... */
#define evutil_timersub(tvp, uvp, vvp) /* ... */
```
这些宏对前两个参数做加法或减法（分别地），并把结果存在第三个参数中。

接口
```c++
#define evutil_timerclear(tvp) /* ... */
#define evutil_timerisset(tvp) /* ... */
```
将 timeval 置 0。检查 timeval 是否设置了值，非 0 返回真，否则返回假。

接口
```c++
int evutil_gettimeofday(struct timeval *tv, struct timezone *tz);
```
evutil_gettimeofday 函数将 tv 设置成当前时间。 tz 参数没有使用。


# R6：缓冲事件之概念和基础

大多数情况下，除了响应事件外，程序还要执行一定数量的数据缓冲。例如，当我们需要写数据，通常的模式：
- 决定要向连接写一些数据，将数据放到缓冲区内。
- 等待连接变成可写的。
- 尽可能多地写数据。
- 记录写了多少数据，如果还有更多的数据需要写，等待连接再次可写。

这种缓冲 IO 模式很常见，Libevent 为此提供了一种通用机制。缓冲事件（ bufferevent ）由
底层传输接口（例如套接字），读缓冲以及写缓冲组成。与底层传输接口可读或可写时回调的普
通事件不同，缓冲事件在写或读了足够数据后调用用户提供的回调函数。

## 缓冲事件和缓冲区
每一个缓冲事件有一个输入缓冲区和一个输出缓冲区，都是缓冲区类型的。当需要向缓冲事件写
数据时，会将其加入输出缓冲区；当需要向缓冲事件读数据时，从输入缓冲区中抽取数据。

## 回调和阈值
缓冲事件有两个数据相关回调：一个读回调和一个写回调。默认情况下，只要从底层传输接口读
取了数据就会调用读回调，只要足够多的数据从输出缓冲区清空到底层传输接口就会调用写回
调。然而您可以通过调整读写 "阈值（watermarks，或翻译成水位）" 来覆盖这些函数的行为。

每个缓冲事件有四个阈值：

读低阈
 
 每当缓冲事件的输入缓冲区数据达到或超过该阈值，则发生读回调。默认是 0，因此一旦读缓
冲区内有数据就会调用读回调。

读高阈
 
 如果缓冲事件的输入缓冲区数据达到该阈值，则停止读取。直到有足够的数据被抽取，使得缓
冲区内数据低阈该阈值，才会继续读取。默认是无穷，因此绝对不会因为输入缓冲区内有太多数
据而停止读取。

写低阈
 
 如果输出缓冲区数据达到或低于该阈值，则发生写回调。默认是 0，所以只有输出缓冲区是空
的时候才会调用写回调。

写高阈
 
 缓冲事件不直接使用该值，该阈值在缓冲事件被用作另一个缓冲事件的底层传输接口时具有特
殊含义。

缓冲事件同样拥有 "错误" 或 "事件" 回调，以便在连接关闭或发生错误等情况下通知通知应用发
生了非数据事件。以下是定义的事件标志：

BEV_EVENT_READING
 
 读操作时发生某事件，具体是什么事件请参考其他标志。

BEV_EVENT_WRITING
 
 写操作时发生某事件，具体是什么事件请参考其他标志。

BEV_EVENT_ERROR
 
 缓冲事件操作发生错误，关于错误的更多信息，可以调用 EVUTIL_SOCKET_ERROR() 。

BEV_EVENT_TIMEOUT
 
 缓冲事件超时。

BEV_EVENT_EOF
 
 得到文件结束指示。

BEV_EVENT_CONNECTED
 
 连接请求已完成。

## 缓冲事件选项标志
在创建缓冲事件时可以通过一个或多个标志改变其行为，可识别的标志有：

BEV_OPT_CLOSE_ON_FREE
 
 当缓冲事件释放时，关闭其底层传输。这会关闭底层套接字，底层缓冲事件等等。

BEV_OPT_THREADSAFE
 
 自动为缓冲事件分配锁，因此是多线程安全的。

BEV_OPT_DEFER_CALLBACKS
 
 设置该标志后，缓冲事件会延迟回调。

BEV_OPT_UNLOCK_CALLBACKS
 
 默认情况下，当缓冲事件被设置成线程安全，任何用户提供的回调函数被调用时都会上锁。设
置此标志后 Libevent 会在调用您的回调函数时释放锁。

## 使用基于套接字的缓冲事件
最易于使用的缓冲事件是基于套接字的。基于套接字的缓冲事件使用 Libevent 底层事件机制来检测网络套接字是否可读且/或可写，并且使用底层网络调用（如 readv ， writev ， WSASend或 WSARecv ）来传输和接收数据。

## 创建基于套接字的缓冲事件
可以通过 bufferevent_socket_new() 来创建一个基于套接字的缓冲事件：

接口
```c++
struct bufferevent *bufferevent_socket_new(
    struct event_base *base,
    evutil_socket_t fd,
    enum bufferevent_options options);
```
base 参数是一个 event_base ， options 是缓冲事件选项（ BEV_OPT_CLOSE_ON_FREE 等）组成的比特字段， fd 是一个可选的套接字文件描述符。如果您想稍后设置文件描述符，则可以把fd 先设成 -1。

技巧：确保提供给 bufferevent_socket_new 的套接字是非阻塞的，Libevent 为此提供了方
便的 evutil_make_socket_nonblocking 。

## 在基于套接字的缓冲事件上建立连接
如果缓冲事件的套接字还没有连接，您可以创建一个新的连接。

接口
```c++
int bufferevent_socket_connect(struct bufferevent *bev,
    struct sockaddr *address, int addrlen);
```
address 和 addrlen 参数与标准函数 connect() 一致。如果缓冲事件还没有设置套接字，调用该函数会为其分配一个新的非套接字。

如果缓冲事件已经设置了套接字，调用 bufferevent_socket_connect() 会告知 Libevent 该套接字还没有建立连接，直到连接建立完成前都不会有读写操作。

在建立连接之前将数据添加进输出缓冲区是合法的。

该函数在成功建立连接时返回 0，发生错误时返回 -1。

注意如果使用 bufferevent_socket_connect() 建立连接，则只会收到 BEV_EVENT_CONNECTED
的事件；如果手动调用 connect() ，则会收到可写事件。

## 通用 bufferevent 操作
释放 bufferevent

接口
```c++
void bufferevent_free(struct bufferevent *bev);
```
该函数用于释放缓冲事件。缓冲事件内部有索引计数，因此如果释放时还有等待中的延迟回调，会先回调再释放。

然而， bufferevent_free() 函数会尝试尽快释放缓冲事件。如果此时缓冲事件中有数据正在等待写（如写入文件或套接字），则在缓冲事件释放之前，缓冲区可能不会刷新（即待写数据丢失）。

如果设置了 BEV_OPT_CLOSE_ON_FREE 标志，并且该缓冲事件带有一个套接字或底层缓冲事件作为其传输接口，则传输接口会在缓冲事件释放时一并关闭。

## 回调、阈值和启用开关
接口
```c++
typedef void (*bufferevent_data_cb)(struct bufferevent *bev, void *ctx);
typedef void (*bufferevent_event_cb)(struct bufferevent *bev,
    short events, void *ctx);
void bufferevent_setcb(struct bufferevent *bufev,
    bufferevent_data_cb readcb, bufferevent_data_cb writecb,
    bufferevent_event_cb eventcb, void *cbarg);
void bufferevent_getcb(struct bufferevent *bufev,
    bufferevent_data_cb *readcb_ptr,
    bufferevent_data_cb *writecb_ptr,
    bufferevent_event_cb *eventcb_ptr,
    void **cbarg_ptr);
```
bufferevent_setcb() 函数改变缓冲事件的一个或多个回调函数。 readcb ， writecb 和eventcb 分别在（缓冲事件）已经读取足够的数据（输入缓冲数据高于阈值），已经写入足够的数据（输出缓冲数据低于阈值）以及发生某些事件时回调。回调函数的第一个参数是触发了事件的缓冲事件，最后一个参数是调用 bufferevent_setcb() 时设置的 cbarg 参数：您可以通过这个参数向回调函数传递数据。事件回调（ bufferevent_event_cb ）的参数 events 是事件标志的比特掩码，参考上方的回调和阈值。

可以通过传递 NULL 来禁用某个回调。注意所有的回调函数都共享同一个 cbarg 值，改变这个值会影响所有回调函数。

您可以通过向 bufferevent_getcb() 传递指针以获取缓冲事件当前设置的回调函数

接口
```c++
void bufferevent_enable(struct bufferevent *bufev, short events);
void bufferevent_disable(struct bufferevent *bufev, short events);
short bufferevent_get_enabled(struct bufferevent *bufev);
```
您可以开启或关闭缓冲事件的 EV_READ ， EV_WRITE 或 EV_READ|EV_WRITE 事件。如果没有开启读和写，缓冲事件不会尝试读写数据。

输出缓冲为空时没有必要关闭写事件：缓冲事件会自动停止写数据，并在有数据可写时重新开始写。

同样，输入缓冲区到达读高阈时无需关闭读事件：缓冲事件会自动停止读取，并在有足够空间时继续读取。

默认情况下，刚创建的缓冲事件开启了写事件，但没有开启读事件。

可以通过调用 bufferevent_get_enabled() 获取缓冲事件当前开启的事件。

接口
```c++
void bufferevent_setwatermark(struct bufferevent *bufev, short events,
    size_t lowmark, size_t highmark);
```
bufferevent_setwatermark() 函数会调整一个缓冲事件的读阈值或写阈值，或两者都调整。（如果 events 设置了 EV_READ ，则会调整读阈值，如果 events 设置了 EV_WRITE ，则会调整写阈值。）

高阈设置成 0 意味着 "无穷"。

## 处理缓冲事件的数据
接口
```c++
struct evbuffer *bufferevent_get_input(struct bufferevent *bufev);
struct evbuffer *bufferevent_get_output(struct bufferevent *bufev);
```
这两个函数是非常强大的基础接口：它们分别返回输入和输出缓冲区。

注意程序只能移除输入缓冲区中的数据（而非添加数据），也只能向输出缓冲区中添加数据（而非移除数据）。

如果写操作因数据量太少而停止（或读操作因数据量太多而停止），向输出缓冲区添加数据（或移除输入缓冲区中的数据）会自动重启操作。（这里指的应当是输出缓冲区内的数据量太少，需要等待应用向缓冲区写入一定量的数据再一起写入底层传输接口，或输入缓冲区数据太多，需等待应用读取并移除一定量的数据再从底层传输接口读取数据。）

接口
```c++
int bufferevent_write(struct bufferevent *bufev,
    const void *data, size_t size);
int bufferevent_write_buffer(struct bufferevent *bufev,
    struct evbuffer *buf);
```
这些函数向缓冲事件的输出缓冲区写入数据。调用 bufferevent_write() 将 data 起的 size字节数据添加至输出缓冲区的尾部。调用 bufferevent_write_buffer() 将 buf 的所有数据添加至输出缓冲区的尾部。两者都在成功时返回 0，发生错误时返回 -1。

接口
```c++
size_t bufferevent_read(struct bufferevent *bufev, void *data, size_t size);
int bufferevent_read_buffer(struct bufferevent *bufev,struct evbuffer *buf);
```
这些函数从缓冲事件的输入缓冲区移除数据。 bufferevent_read() 函数从输入缓冲区移除size 字节数据，存入 data ，返回实际移动的字节数。 bufferevent_read_buffer() 函数将输入缓冲区的数据全部移入 buf ，成功返回 0，失败返回 -1。注意对于 bufferevent_read() ，内存块 data 必须有足够的空间存储 size 字节数据。

## 刷新缓冲事件
接口
```c++
int bufferevent_flush(struct bufferevent *bufev,
    short iotype, enum bufferevent_flush_mode state);
```
刷新缓冲事件会告知缓冲事件忽略其他限制，强制从底层传输接口读取或写入尽可能多的数据。该函数的功能细节与缓冲事件的类型有关。

iotype 参数应当是 EV_READ ， EV_WRITE 或 EV_READ|EV_WRITE ，以指明应当处理读还是写，或者二者均处理。 state 参数可能是 BEV_NORMAL ， BEV_FLUSH 或 BEV_FINISHED 中的一个。 BEV_FINISHED 表示应当告知另一端没有更多数据需要发送了， BEV_NORMAL 和BEV_FLUSH 之间的区别则与缓冲事件的类型有关。

bufferevent_flush() 函数失败时返回 -1，没有数据需要刷新时返回 0，刷新了数据则返回1。

## 类型特定的缓冲事件函数
这些函数并不支持所有类型的缓冲事件。

接口
```c++
int bufferevent_priority_set(struct bufferevent *bufev, int pri);
int bufferevent_get_priority(struct bufferevent *bufev);
```
该函数（ bufferevent_priority_set ）将 bufev 的优先级调整为 pri 。

该函数成功返回 0，失败返回 -1，且只对套接字类型的缓冲事件有效。

接口
```c++
int bufferevent_setfd(struct bufferevent *bufev, evutil_socket_t fd);
evutil_socket_t bufferevent_getfd(struct bufferevent *bufev);
```
这些函数为文件类型缓冲事件设置或返回文件描述符。只有套接字类型的缓冲事件支持 setfd()
。这两个函数都在失败时返回 -1， setfd() 成功时返回 0.

接口
```c++
struct event_base *bufferevent_get_base(struct bufferevent *bev);
```
该函数返回缓冲事件的 event_base 

接口
```c++
struct bufferevent *bufferevent_get_underlying(struct bufferevent *bufev);
```
该函数返回底层作为传输接口的缓冲事件。

## 手动锁定和解锁
与缓冲区一样，有时您希望确保对缓冲事件的许多操作是原子的。Libevent 提供了一些可以手动锁定和解锁缓冲事件的函数。

接口
```c++
void bufferevent_lock(struct bufferevent *bufev);
void bufferevent_unlock(struct bufferevent *bufev);
```
注意如果创建缓冲事件时没有设置 BEV_OPT_THREADSAFE 或没有启用 Libeven 对多线程的支持，则锁定缓冲事件是没有任何效果的。

使用该函数锁定缓冲事件会一并锁定其关联的缓冲区。这些函数是可递归的：持有锁时再次对缓冲事件上锁是安全的。当然，您必须要对每一次上锁操作调用解锁。

# R7：缓冲 IO 实用功能——缓冲区
Libevent 的缓冲区（ evbuffer ）功能实现了一个字节队列，优化了向尾部添加数据以及从头部移除数据的功能。

缓冲区通常用作缓冲网络 IO 的 "缓冲" 部分，缓冲区不提供调度 IO 或在准备就绪时触发 IO：这是缓冲事件该做的。

## 创建或释放缓冲区
接口
```c++
struct evbuffer *evbuffer_new(void);
void evbuffer_free(struct evbuffer *buf);
```
evbuffer_new() 分配并返回一个空的缓冲区， evbuffer_free() 删除缓冲区及其所有组件

## 缓冲区和线程安全
接口
```
int evbuffer_enable_locking(struct evbuffer *buf, void *lock);
void evbuffer_lock(struct evbuffer *buf);
void evbuffer_unlock(struct evbuffer *buf);
```
默认情况下，多线程同时访问缓冲区是不安全的。如果需要多线程访问，可以对缓冲区调用
evbuffer_enable_locking() 。如果 lock 参数是 NULL ，Libevent 调用先前通过
evthread_set_lock_creation_callback 设置的锁创建函数分配一个新锁，否则使用该参数作为锁。

## 检查缓冲区
接口
```c++
size_t evbuffer_get_length(const struct evbuffer *buf);
```
该函数以字节为单位返回存储在缓冲区中的数据长度。

接口
```c++
size_t evbuffer_get_contiguous_space(const struct evbuffer *buf);
```
该函数返回在缓冲区头部连续存储的数据的字节数。缓冲区中的数据可能存储在好几个不连续的内存块，这个函数返回缓冲区当前第一个区块存储的字节数。

## 向缓冲区添加数据：基础
接口
```c++
int evbuffer_add(struct evbuffer *buf, const void *data, size_t datlen);
```
该函数把 data 中的 datlen 字节长度数据添加到 buf 的末端，成功返回 0，失败返回 -1。

接口
``` c++
int evbuffer_add_printf(struct evbuffer *buf, const char *fmt, ...);
int evbuffer_add_vprintf(struct evbuffer *buf, const char *fmt, va_list ap);
```
这些函数向 buf 的尾部添加格式化数据，格式化参数和其他参数的处理分别和 C 库的 printf
以及 vprintf 一致。这些函数返回向缓冲区中添加的字节数。

## 将数据转移到另一个缓冲区
接口
```c++
int evbuffer_add_buffer(struct evbuffer *dst, struct evbuffer *src);
int evbuffer_remove_buffer(struct evbuffer *src, struct evbuffer *dst,
    size_t datlen);
```
evbuffer_add_buffer() 函数将 src 的所有数据移动到 dst 中，成功返回 0，失败返回-1

evbuffer_remove_buffer() 函数将 src 中 datlen 字节数据移动到 dst 的末端，并且尽可
能避免拷贝。如果可移动的数据小于 datlen ，则会移动所有数据。该函数返回移动的字节数。

## 向缓冲区头部添加数据
接口
```c++
int evbuffer_prepend(struct evbuffer *buf, const void *data, size_t size);
int evbuffer_prepend_buffer(struct evbuffer *dst, struct evbuffer* src);
```
这些函数行为与 evbuffer_add() 和 evbuffer_add_buffer() 相似，不同的是将数据添加进目标缓冲区的头部。

## 从缓冲区移除数据
接口
```c++
int evbuffer_drain(struct evbuffer *buf, size_t len);
int evbuffer_remove(struct evbuffer *buf, void *data, size_t datlen);
```
evbuffer_remove() 函数拷贝并移除 buf 的前 datlen 字节数据到 data 中。如果缓冲区中的数据少于 datlen 字节，则会拷贝所有数据。该函数失败时返回 -1，否则返回拷贝的字节数。

evbuffer_drain() 函数行为与 evbuffer_remove() 函数类似，但是不会拷贝数据：该函数仅仅移除缓冲区首部数据，成功返回 0，失败返回 -1。

## 从缓冲区拷贝数据
有时可能需要拷贝缓冲区首部数据，但不希望移除这些数据。例如，您可能希望查看某种类型的完整记录是否已经到达，而不移除任何数据（如 evbuffer_remove 所做的），或者重排内部布局（如 evbuffer_pullup() 所做的）。

接口
```c++
ev_ssize_t evbuffer_copyout(struct evbuffer *buf, void *data, size_t datlen);
ev_ssize_t evbuffer_copyout_from(struct evbuffer *buf, const struct evbuffer_ptr *pos,
    void *data_out, size_t datlen);
```
evbuffer_copyout() 函数的行为与 evbuffer_remove() 类似，只不过不会将数据从缓冲区移
除。即，拷贝 buf 的前 datlen 字节数据到到 data 。如果数据量小于 datlen 字节，则会拷
贝所有字节。该函数失败返回 -1，成功返回拷贝的字节数。

evbuffer_copyout_from() 函数行为与 evbuffer_copyout() 类似，只不过是从缓冲区指针
pos 开始拷贝，而非首部开始。

如果从缓冲区拷贝数据太慢，请使用 evbuffer_peek() 替代。

示例
```c++
#include <event2/buffer.h>
#include <event2/util.h>
#include <stdlib.h>
#include <stdlib.h>
int get_record(struct evbuffer *buf, size_t *size_out, char **record_out)
{
    /* 假设我们讨论的是某种协议，其中包含一个按网络顺序排列的 4 字节大小的字段，
    后面跟着字节数。如果记录完整，我们返回 1 并设置 'out' 字段；如果记录不
    完整，则返回 0；发生错误则返回 -1。 */
    size_t buffer_len = evbuffer_get_length(buf);
    ev_uint32_t record_len;
    char *record;

    if (buffer_len < 4)
        return 0; /* size 字段还未到达。 */

    /* 使用 evbuffer_copyout，因此 size 字段还保留在缓冲区中。 */
    evbuffer_copyout(buf, &record_len, 4);
    /* 将 len_buf 转换成主机字节序 */
    record_len = ntohl(record_len);
    if (buffer_len < record_len + 4)
        return 0; /* 记录还不完整 */

    /* 好了，现在我们可以移除记录。 */
    record = malloc(record_len);
    if (record == NULL)
        return -1;

    evbuffer_drain(buf, 4);
    evbuffer_remove(buf, record, record_len);

    *record_out = record;
    *size_out = record_len;
    return 1;
}
```

## 面向行的输入
接口
```c++
enum evbuffer_eol_style {
    EVBUFFER_EOL_ANY,
    EVBUFFER_EOL_CRLF,
    EVBUFFER_EOL_CRLF_STRICT,
    EVBUFFER_EOL_LF,
    EVBUFFER_EOL_NUL
};
char *evbuffer_readln(struct evbuffer *buffer, size_t *n_read_out,
    enum evbuffer_eol_style eol_style);
```

很多网络协议使用基于行的格式。 evbuffer_readln() 函数从缓冲区头部提取一行并放置在新
分配的，以 NUL 结尾的字符串中。如果 n_read_out 不是 NULL ，则 *n_read_out 会被设置
成字符串的字节数。如果无法读取一整行，该函数返回 NULL 。行结束符不会被拷贝到字符串
中。

evbuffer_readln() 函数可以识别 4 种行结束格式：

EVBUFFER_EOL_LF
 
 行尾是一个换行符。（也就是 "\n"，ASCII 码是 0x0A。）

EVBUFFER_EOL_CRLF_STRICT
 
 行尾是一个回车符接着一个换行符。（也就是 "\r\n"，ASCII 码是 0x0D 0x0A）。

EVBUFFER_EOL_CRLF

 行尾是一个可选的回车符，接着一个换行符。（换句话说，既可以是 "\r\n"，也可以是
"\n"。）这种格式对分析基于文本的网络协议很有用，因为标准通常要求以 "\r\n" 结尾，而不遵
循标准的的客户端有时只使用 "\n"。

EVBUFFER_EOL_ANY
 
 行尾可以是任意数量的回车符和任意数量的换行符的任意序列。这种格式并不很有用，其存在
只是为了向后兼容。

EVBUFFER_EOL_NUL
 
 行尾是一个值为 0 的字节，即 ASCII NUL。  
（注意如果使用 event_set_mem_functions() 覆盖默认 malloc ， evbuffer_readln 返回的
字符串会由指定的 malloc 函数分配。

## 无拷贝检查数据
有时可能需要在不拷贝的前提下读取缓冲区的数据（如 evbuffer_copyout() ），并且不重排缓
冲区内部内存（如 evbuffer_pullup() ）。有时可能需要查看缓冲区中间部分的数据。

接口
```c++
struct evbuffer_iovec {
void *iov_base;
size_t iov_len;
};
int evbuffer_peek(struct evbuffer *buffer, ev_ssize_t len,
    struct evbuffer_ptr *start_at,
    struct evbuffer_iovec *vec_out, int n_vec);
```
调用 evbuffer_peek() 时，需要提供一个 evbuffer_iovec 类型的数组 vec_out 。该数组的
长度是 n_vec 。该函数会将数组中的元素设置成指向缓冲区内存区块的指针（ iov_base ），并
将 iov_len 设置成对应区块的长度。

如果 len 小于 0， evbuffer_peek() 会尝试填充 vec_out 中的所有元素。否则，该函数会填
充至 vec_out 所有元素均被使用，或缓冲区中至少 len 比特数据可见。如果该函数可以给出
请求的所有数据，则返回 evbuffer_iovec 中实际使用的元素数量。否则，函数返回给出所有请
求数据所需的元素个数。

如果 ptr 是 NULL ， evbuffer_peek() 从缓冲区首部开始，否则从 ptr 开始。

## 使用缓冲区的网络 IO
Libevent 中缓冲区最常见的用法就是网络 IO。使用缓冲区的网络 IO 接口是：
接口
```c++
int evbuffer_write(struct evbuffer *buffer, evutil_socket_t fd);
int evbuffer_write_atmost(struct evbuffer *buffer, evutil_socket_t fd,
        ev_ssize_t howmuch);
int evbuffer_read(struct evbuffer *buffer, evutil_socket_t fd, int howmuch);
```
evbuffer_read() 函数从套接字 fd 读取最多 howmuch 字节数据到 buffer 中。该函数成功
返回读取的字节数，遇到 EOF 返回 0，发生错误返回 -1。注意错误可能指示非阻塞操作不能立
即成功，您应当检查错误码是否为 EAGAIN （在 Windows 上是 WSAEWOULDBLOCK ）。如果
howmuch 为负数，则 evbuffer_read() 会尝试自己猜测大小。

evbuffer_write_atmost() 函数尝试向套接字 fd 至多写入 buffer 的前 howmuch 字节数
据。该函数成功返回写入的字节数，失败返回 -1。与 evbuffer_read() 一样，需要检查错误码
以判断是否真的发生了错误，或者仅仅是非阻塞 IO 不能立刻完成。如果提供一个负的 howmuch
，该函数会尝试把缓冲区中所有内容写进套接字。

## 缓冲区和回调
缓冲区用户可能频繁地需要知道是否有数据添加进缓冲区或从缓冲区移除，为了支持这些功能，Libevent 提供了一个通用的缓冲区回调机制。

接口
```c++
struct evbuffer_cb_info {
        size_t orig_size;
        size_t n_added;
        size_t n_deleted;
};
typedef void (*evbuffer_cb_func)(struct evbuffer *buffer,
    const struct evbuffer_cb_info *info, void *arg);
```

缓冲区回调会在有数据添加进缓冲区或从缓冲区移除时调用。回调函数接收缓冲区，指向
evbuffer_cb_info 结构体的指针以及用户提供的 arg 作为参数。 evbuffer_cb_info 结构体
的 orig_size 字段记录了缓冲区发生变化之前存储的数据的字节数； n_added 字段记录了添加
进缓冲区的字节数； n_deleted 记录了从缓冲区移除的字节数。

接口
```c++
struct evbuffer_cb_entry;
struct evbuffer_cb_entry *evbuffer_add_cb(struct evbuffer *buffer,
    evbuffer_cb_func cb, void *cbarg);
```
evbuffer_add_cb() 函数为缓冲区注册一个回调函数，并且返回指向可以用于之后代码中标识
特定回调实例的不透明指针。 cb 参数是将被调用的函数， cbarg 是用户向回调函数提供的参数。

可以为一个缓冲区添加多个回调函数，添加新的回调不会移除旧的回调。

## 通过引用将缓冲区添加到另一个缓冲区
可以通过引用将缓冲区添加到另一个缓冲区：并非将缓冲区的内容移动到另一个缓冲区中，而是
将缓冲区的引用提供给另一个，看起来就像将数据拷贝过去了一样。

接口
```c++
int evbuffer_add_buffer_reference(struct evbuffer *outbuf,
    struct evbuffer *inbuf);
```
evbuffer_add_buffer_reference() 函数的行为与将 outbuf 所有内容拷贝至 inbuf 类似，
但是不会进行任何不必要的拷贝。该函数成功返回 0，失败返回 -1。

注意，对 inbuf 内容的后续更改不会反映到 outbuf 中：该函数引用的是缓冲区的当前内容，
而不是缓冲区本身。

## 限定缓冲区只能添加或删除数据
接口
```c++
int evbuffer_freeze(struct evbuffer *buf, int at_front);
int evbuffer_unfreeze(struct evbuffer *buf, int at_front);
```
可以通过这些函数暂时禁止缓冲区头部或尾部的改动。缓冲事件的代码内部使用了这些函数以避
免输出缓冲区头部或输入缓冲区尾部的意外改动。

# R8：接受 TCP 连接的监听器
监听器（ evconnlistener ）机制提供了监听并接受 TCP 连接的机制。

## 创建或释放监听器
接口
```c++
struct evconnlistener *evconnlistener_new(struct event_base *base,
    evconnlistener_cb cb, void *ptr, unsigned flags, int backlog,
    evutil_socket_t fd);
struct evconnlistener *evconnlistener_new_bind(struct event_base *base,
    evconnlistener_cb cb, void *ptr, unsigned flags, int backlog,
    const struct sockaddr *sa, int socklen);
void evconnlistener_free(struct evconnlistener *lev);
```
两个 evconnlistener_new*() 函数都会申请并返回一个新的连接监听对象。连接监听器使用
event_base 检测给定的监听套接字上是否有新的 TCP 连接。当出现新的连接时，将调用给定的
回调函数。

在这两个函数中， base 参数是监听器用于监听连接的 event_base 。 cb 函数是接收到新的连
接时调用的回调函数，如果 cb 是 NULL ，监听器被认为是无效的，知道设置了回调函数。 ptr
指针会被传递给回调函数。 flag 参数控制回调函数的行为，下面会有详细介绍。 backlog 参数
控制网络栈同时允许的最大挂起连接数量，更多信息请参考您的系统中 listen() 函数的相关文
档。如果 backlog 是负的，Libevent 选择一个合适的值；如果是 0，Libevent 会假定您在提
供套接字 fd 之前已经调用了 listen() 。

这两个函数的区别之处在于如何设置监听套接字。 evconnlistener_new() 函数假定您已经将套
接字绑定到了需要监听的端口，并通过 fd 传递套接字。如果想要 Libevent 分配并绑定套接
字，调用 evconnlistener_new_bind() 并传递需要绑定的 sockaddr 及其长度

```
提示：[使用 evconnlistener_new 时，请确保使用 evutil_make_socket_nonblocking 或者
其他手动方式将套接字设置成非阻塞模式。如果套接字处于阻塞模式，可能会发生未定义的行
为。]
```

要释放一个连接监听器，将其传递给 evconnlistener_free() 即可。

## 可识别的标志
以下是可以通过 evconnlistener_new() 函数的 flags 参数传递的标志，您可以传递传递以下标志的任何一个或组合。

LEV_OPT_LEAVE_SOCKETS_BLOCKING

 默认情况下，连接监听器接受的新连接都会被设置成非阻塞以便在 Libevent 中使用。如果不想要该行为可以设置此标志。

LEV_OPT_CLOSE_ON_FREE
 
 如果设置了该标志，释放连接监听器时会关闭其底层套接字。

LEV_OPT_CLOSE_ON_EXEC

 如果设置了此标志，连接监听器会为底层套接字设置 close-on-exec 标志，更多信息请参考平台关于 fcntl 和 FD_CLOEXEC 相关文档。

LEV_OPT_REUSEABLE
 
 在一些平台中的默认配置下，一旦监听套接字关闭，该套接字端口短时间内不能重复使用。设
置此标志会让 Libevent 将套接字设置成可重用，即套接字一旦关闭，其他套接字可以在相同端口上开启监听。

LEV_OPT_THREADSAFE

 为监听器分配锁，使其在多线程使用中安全，于 Libevent 2.0.8-rc 新引入。

LEV_OPT_DISABLED

 将监听器初始化为禁用状态，而非启用。可以通过手动调用 evconnlistener_enable() 启用

LEV_OPT_DEFERRED_ACCEPT

 如果可能的话，通知内核在接收到套接字上的数据并准备好读取之前，不要宣布套接字已被接
受。如果您的协议不是以客户端传输数据开始的，则不要启用该标志，因为在这种情况下内核可
能永远不会通知程序已经建立连接。并不是所有的操作系统都支持这个选项，如果不支持，则该
标志没有任何效果。

## 连接监听器回调
接口
```c++
typedef void (*evconnlistener_cb)(struct evconnlistener *listener,
    evutil_socket_t sock, struct sockaddr *addr, int len, void *ptr);
```
当接收到新的连接时，提供的回调函数将会被调用。 listener 参数是接收连接请求的连接监听
器， sock 参数是新的套接字， addr 和 len 参数是新连接的地址和地址长度， ptr 参数是调
用 evconnlistener_new() 时用户提供的指针。

## 开启和关闭监听器
接口
```c++
int evconnlistener_disable(struct evconnlistener *lev);
int evconnlistener_enable(struct evconnlistener *lev);
```
这些函数可以暂时关闭或重新开启监听器。

## 调整监听器回调
接口
```c++
void evconnlistener_set_cb(struct evconnlistener *lev,
    evconnlistener_cb cb, void *arg);
```
该函数为监听器调整回调函数及其参数

## 检查监听器

接口
```c++
evutil_socket_t evconnlistener_get_fd(struct evconnlistener *lev);
struct event_base *evconnlistener_get_base(struct evconnlistener *lev);
```
这些函数分别返回监听器关联的套接字和 event_base

## 检测错误
您可以通过设置错误回调以在 accept() 调用失败时获取相关信息，这在面临一个不解决就会造成整个线程锁定的错误时是很重要的。

接口
``` c++
typedef void (*evconnlistener_errorcb)(struct evconnlistener *lis, void *ptr);
void evconnlistener_set_error_cb(struct evconnlistener *lev,
evconnlistener_errorcb errorcb);
```

如果使用 evconnlistener_set_error_cb() 为监听器设置错误回调，每次监听器发生错误时都
会调用该回调。该回调会接收监听器作为第一个参数， evconnlistener_new() 的 ptr 作为第
二个参数。

