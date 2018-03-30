### 协程的定义
关于协程，你可能看的最多的就是这样一句话“协程就是用户态的线程”。

要理解是什么是“用户态的线程”，必然就要先理解什么是“内核态的线程”。 内核态的线程是由操作系统来进行调度的，在切换线程上下文时，要先保存上一个线程的上下文，然后执行下一个线程，当条件满足时，切换回上一个线程，并恢复上下文。 协程也是如此，只不过，用户态的线程不是由操作系统来调度的，而是由程序员来调度的，是在用户态的。

### Swoole协程概述
Swoole的协程实现实际上是基于PHP的Yield机制的。通过Yield可以保存一个PHP function的现场。再结合Swoole提供的EventLoop，在IO发起时存储function现场，IO完成时恢复function现场。底层实现了一个调度器完成php function stack的管理。主要用与高并发IO的场景，同时并发执行大量IO操作，目前支持 Redis，MySQL，TCP/UDP Client，HttpClient 4种IO操作。

### Swoole源码分析
swoole_coroutine.c中定义了一个全局变量COROG，它的结构为：
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
由此可见，该变量是用于存储协程的初始化信息。此外，swoole_coroutine.c的头部还定义了一个很重要的宏叫做SWCC，用于保存当前协程的详细信息，我们知道Swoole为每个协程都分配了空间，用于保存协程切换时的状态信息。进行协程切换时会自动保存Zend VM的内存状态（主要是EG全局内存和vm stack）。那么所谓的状态信息就是保存于SWCC这个宏当中，回调函数执行完后释放。

接下来我们介绍swoole协程中几个比较重要的动作

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
    //协程切换基于setjmp,longjmp（不了解这两个函数的可以把它理解成goto）
    if (!setjmp(*swReactorCheckPoint))
    {
        zend_execute_ex(call);
        coro_close(TSRMLS_C);
        swTraceLog(SW_TRACE_COROUTINE, "[CORO_END] Create the %d coro with stack. heap size: %zu", COROG.coro_num, zend_memory_usage(0));
        coro_status = CORO_END;
    }
    else
    {
        //让出协程
        coro_status = CORO_YIELD;
    }
    COROG.require = 0;
    return coro_status;
}
```
#### 让出协程
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

#### todo
