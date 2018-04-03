### 协程的定义
关于协程，你可能看的最多的就是这样一句话“协程就是用户态的线程”。

要理解是什么是“用户态的线程”，必然就要先理解什么是“内核态的线程”。 内核态的线程是由操作系统来进行调度的，在切换线程上下文时，要先保存上一个线程的上下文，然后执行下一个线程，当条件满足时，切换回上一个线程，并恢复上下文。 协程也是如此，只不过，用户态的线程不是由操作系统来调度的，而是由程序员来调度的，是在用户态的。

### Swoole协程概述
Swoole的协程实现实际上是基于PHP的Yield机制的。通过Yield可以保存一个PHP function的现场。再结合Swoole提供的EventLoop，在IO发起时存储function现场，IO完成时恢复function现场。底层实现了一个调度器完成php function stack的管理。主要用于高并发IO的场景，同时并发执行大量IO操作，目前支持 Redis，MySQL，TCP/UDP Client，HttpClient 4种IO操作。

### Swoole源码分析
swoole_coroutine.c的头部定义了一个全局变量COROG，它的结构为：
```c
typedef struct _coro_global
{
    uint32_t coro_num;
    uint32_t max_coro_num;
    uint32_t stack_size;
    zend_vm_stack origin_vm_stack;
...
    zval *origin_vm_stack_top;
    zval *origin_vm_stack_end;
    zval *allocated_return_value_ptr;
...
} coro_global;
```
该变量是用于存储协程的基本信息如开启的协程数量。此外，swoole_coroutine.c中还定义了一个很重要的宏叫做SWCC，用于保存当前协程的详细信息，我们知道Swoole为每个协程都分配了空间，用于保存协程切换时的状态信息。进行协程切换时会自动保存Zend VM的内存状态（主要是EG全局内存和vm stack）。那么所谓的状态信息就是保存于SWCC这个宏当中，回调函数执行完后释放。

接下来我们介绍swoole协程中几个比较重要的动作。

#### 创建协程
```c
int sw_coro_create(zend_fcall_info_cache *fci_cache, zval **argv, int argc, zval *retval, void *post_callback, void* params)
{
    //为回调函数的执行做一些准备工作
    ...
    COROG.allocated_return_value_ptr = retval;
    EG(current_execute_data) = NULL;
    zend_init_execute_data(call, op_array, retval);

    ++COROG.coro_num;
    COROG.current_coro->cid = cid;
    COROG.current_coro->start_time = time(NULL);
    COROG.current_coro->function = NULL;
    COROG.current_coro->post_callback = post_callback;
    COROG.current_coro->post_callback_params = params;
    COROG.require = 1;

    int coro_status;
    //协程切换基于setjmp,longjmp（不了解这两个函数的可以类比理解成goto）
    if (!setjmp(*swReactorCheckPoint))
    {
        zend_execute_ex(call);
        coro_close(TSRMLS_C);
        swTraceLog(SW_TRACE_COROUTINE, "[CORO_END] Create the %d coro with stack. heap size: %zu", COROG.coro_num, zend_memory_usage(0));
        coro_status = CORO_END;
    }
    else
    {
        //让出协程cpu执行权
        coro_status = CORO_YIELD;
    }
    COROG.require = 0;
    return coro_status;
}
```
#### 让出执行权
```c
sw_inline void coro_yield()
{
    SWOOLE_GET_TSRMLS;
#if PHP_MAJOR_VERSION >= 7
    EG(vm_stack) = COROG.origin_vm_stack;
    EG(vm_stack_top) = COROG.origin_vm_stack_top;
    EG(vm_stack_end) = COROG.origin_vm_stack_end;
#else
    EG(argument_stack) = COROG.origin_vm_stack;
    EG(current_execute_data) = COROG.origin_ex;
#endif
    //和setjmp对应
    longjmp(*swReactorCheckPoint, 1);
}
```
#### 保存协程上下文信息
```c
sw_inline php_context *sw_coro_save(zval *return_value, php_context *sw_current_context)
{
    //将协程上下文信息保存于上面提及的SWCC这个宏
    SWCC(current_coro_return_value_ptr) = return_value;
    SWCC(current_execute_data) = EG(current_execute_data);
    SWCC(current_vm_stack) = EG(vm_stack);
    SWCC(current_vm_stack_top) = EG(vm_stack_top);
    SWCC(current_vm_stack_end) = EG(vm_stack_end);
    SWCC(current_task) = COROG.current_coro;
    SWCC(allocated_return_value_ptr) = COROG.allocated_return_value_ptr;
    return sw_current_context;
}
```
#### 恢复协程
```c
int sw_coro_resume(php_context *sw_current_context, zval *retval, zval *coro_retval)
{
    //从SWCC恢复到EG（关于EG：这是PHP内核中十分重要的一个宏，executor_globals，是Zend执行器相关的全局变量。）
    EG(vm_stack) = SWCC(current_vm_stack);
    EG(vm_stack_top) = SWCC(current_vm_stack_top);
    EG(vm_stack_end) = SWCC(current_vm_stack_end);

    zend_execute_data *current = SWCC(current_execute_data);
    if (ZEND_CALL_INFO(current) & ZEND_CALL_RELEASE_THIS)
    {
        zval_ptr_dtor(&(current->This));
    }
    zend_vm_stack_free_args(current);
    zend_vm_stack_free_call_frame(current);

    EG(current_execute_data) = current->prev_execute_data;
    COROG.current_coro = SWCC(current_task);
    COROG.require = 1;
#if PHP_MINOR_VERSION < 1
    EG(scope) = EG(current_execute_data)->func->op_array.scope;
#endif
    COROG.allocated_return_value_ptr = SWCC(allocated_return_value_ptr);
    if (EG(current_execute_data)->opline->result_type != IS_UNUSED)
    {
        ZVAL_COPY(SWCC(current_coro_return_value_ptr), retval);
    }
    EG(current_execute_data)->opline++;
    int coro_status;
    
    //设置跳转点，方便在执行过程中再遇到异步IO操作，进行跳转。
    if (!setjmp(*swReactorCheckPoint))
    {
        //coro exit
        zend_execute_ex(EG(current_execute_data) TSRMLS_CC);
        coro_close(TSRMLS_C);
        coro_status = CORO_END;
    }
    else
    {
        //coro yield
        coro_status = CORO_YIELD;
    }
    COROG.require = 0;

    if (unlikely(coro_status == CORO_END && EG(exception)))
    {
        sw_zval_ptr_dtor(&retval);
        zend_exception_error(EG(exception), E_ERROR TSRMLS_CC);
    }
    return coro_status;
}
```

#### 协程关闭
```c
sw_inline void coro_close(TSRMLS_D)
{
    //释放内存空间，将COROG.coro_num减一
    swTraceLog(SW_TRACE_COROUTINE, "Close coroutine id %d", COROG.current_coro->cid);
    if (COROG.current_coro->function)
    {
        sw_zval_free(COROG.current_coro->function);
        COROG.current_coro->function = NULL;
    }
    free_cidmap(COROG.current_coro->cid);
    efree(EG(vm_stack));
    efree(COROG.allocated_return_value_ptr);
    EG(vm_stack) = COROG.origin_vm_stack;
    EG(vm_stack_top) = COROG.origin_vm_stack_top;
    EG(vm_stack_end) = COROG.origin_vm_stack_end;
    --COROG.coro_num;
    COROG.current_coro = NULL;
    swTraceLog(SW_TRACE_COROUTINE, "closing coro and %d remained. usage size: %zu. malloc size: %zu", COROG.coro_num, zend_memory_usage(0), zend_memory_usage(1));
}
```
### 实例分析
这是从swoole文档找的一个协程客户端demo：
```php
$client = new Swoole\Coroutine\Client(SWOOLE_SOCK_TCP);
if (!$client->connect('127.0.0.1', 9501, 0.5))
{
    exit("connect failed. Error: {$client->errCode}\n");
}
$client->send("hello world\n");
echo $client->recv();
$client->close();
```
我们分析一下它的行为，源码在swoole_client_coro.c。

首先$client = new Swoole\Coroutine\Client(SWOOLE_SOCK_TCP)实例化对象的时候触发一个构造函数，在这里进行协程栈的初始化：
```c
static PHP_METHOD(swoole_client_coro, __construct)
{
    ...
    //不支持长连
    int client_type = php_swoole_socktype(type);
    if (client_type < SW_SOCK_TCP || client_type > SW_SOCK_UNIX_STREAM)
    {
        swoole_php_fatal_error(E_ERROR, "Unknown client type '%d'.", client_type);
    }

    zend_update_property_long(swoole_client_coro_class_entry_ptr, getThis(), ZEND_STRL("type"), type TSRMLS_CC);
    //init
    swoole_set_object(getThis(), NULL);
    
    //开辟内存空间
    swoole_client_coro_property *client_coro_property = emalloc(sizeof(swoole_client_coro_property));
    bzero(client_coro_property, sizeof(swoole_client_coro_property));
    client_coro_property->iowait = SW_CLIENT_CORO_STATUS_CLOSED;
    swoole_set_property(getThis(), client_coro_property_coroutine, client_coro_property);

    php_context *sw_current_context = emalloc(sizeof(php_context));
    sw_current_context->onTimeout = NULL;
    ...
    sw_current_context->coro_params = *getThis();
    ...
    RETURN_TRUE;
}
```
当客户端发起一个连接请求$client->connect('127.0.0.1', 9501, 0.5)，swoole将保存PHP上下文信息，并让出cpu执行权，待确认连接成功后，触发epoll事件，然后协程切换，恢复PHP上下文信息，返回结果，继续执行PHP代码
```c
static PHP_METHOD(swoole_client_coro, connect)
{
    ...
    swoole_set_object(getThis(), cli);
    ...
    zval *zobject = getThis();
    cli->object = zobject;
    ...
    swoole_client_coro_property *ccp = swoole_get_property(getThis(), 1);
    sw_copy_to_stack(cli->object, ccp->_object);
    ...
    //nonblock async
    if (cli->connect(cli, host, port, timeout, sock_flag) < 0)
    {
        ...
    }
    ...
    php_context *sw_current_context = swoole_get_property(getThis(), 0);
    coro_save(sw_current_context);
    coro_yield();
}
```
client向server发送数据$client->send("hello world\n")：
```c
static PHP_METHOD(swoole_client_coro, send)
{
    ...
    swClient *cli = client_coro_get_ptr(getThis());
    ...
    int ret = cli->send(cli, data, data_len, flags);
    if (ret < 0)
    {
        ...
    }
    ...
}
```
客户端接收数据$client->recv()，recv也是常见的阻塞IO，因此这里同样会保存当前协程的上下文，然后进行yield。待server回复后，触发epoll事件，然后resume。这里有一个超时机制，在设定时间内未能回包则返回false。
```c
static PHP_METHOD(swoole_client_coro, recv)
{
    ...
    swoole_client_coro_property *ccp = swoole_get_property(getThis(), 1);
    if (ccp->iowait == SW_CLIENT_CORO_STATUS_DONE)
    {
        ...
    }
    else if (ccp->iowait == SW_CLIENT_CORO_STATUS_WAIT && ccp->cid != COROG.current_coro->cid)
    {
        ...
    }

    php_context *context = swoole_get_property(getThis(), 0);
    if (timeout > 0)
    {
        php_swoole_check_timer((int) (timeout * 1000));
        ccp->timer = SwooleG.timer.add(&SwooleG.timer, (int) (timeout * 1000), 0, context, client_coro_onTimeout);
    }
    ccp->iowait = SW_CLIENT_CORO_STATUS_WAIT;
    coro_save(context);
    ccp->cid = COROG.current_coro->cid;
    coro_yield();
}
```
$client->close()这一步的操作主要是释放连接、释放内存，把对应的swoole_object置为NULL
```c
static PHP_METHOD(swoole_client_coro, close)
{
    swClient *cli = swoole_get_object(getThis());
    ...
    //Connection error, or short tcp connection.
    swoole_client_coro_property *ccp = swoole_get_property(getThis(), 1);
    ccp->iowait = SW_CLIENT_CORO_STATUS_CLOSED;
    cli->released = 1;
    php_swoole_client_free(getThis(), cli TSRMLS_CC);

    RETURN_TRUE;
}
```
### 总结
以上关于swoole协程的介绍默认是在onReceive、onRequest等回调函数中使用协程客户端。除此之外，swoole还支持用户代码自行创建协程，在不支持协程的回调函数中，可以调用Coroutine::create自行创建协程，swoole2.1以上版本还采用了借鉴Go语言的协程语法糖，其底层调度原理和协程客户端是一致的。
