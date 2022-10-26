##bool __atomic_compare_exchange_n (type *ptr, type *expected, type desired, bool weak, int success_memorder, int failure_memorder)

此内置函数实现原子比较和交换操作。这将\*ptr 的内容与\*expected的内容进行比较。如果相等，则该操作是将desired内容写入*ptr的读-修改-写操作。如果它们不相等，则操作为读取，\*ptr的当前内容将写入\*expected。对于弱compare_exchange而言weak为true，它可能会错误地失败；对于强变量weak为false，它永远不会错误地失败。许多目标只提供强变量，而忽略了参数。如果有疑问，请使用强变量。
如果将desired内容写入了*ptr，则返回true，并根据success_memorder指定的内存顺序影响内存。对于这里可以使用的内存顺序没有限制。
否则，返回false，并根据failure_memorder影响内存。此内存顺序不能是__ATOMIC_RELEASE，也不能是__ATOMIC_ACQ_REL。它也不能是比success_memorder指定的顺序更强的顺序。

# type __sync_lock_test_and_set (type *ptr, type value, ...)

将\*ptr设为value并返回*ptr操作之前的值

# void __sync_lock_release (type *ptr, ...)

将*ptr置0
