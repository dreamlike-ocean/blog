### 在io_uring中设置block的socket是否有意义？

先看IORING_OP_RECVMSG pdef 其支持pollin 

```C++
    [IORING_OP_RECVMSG] = {
        .needs_file     = 1,
        .unbound_nonreg_file    = 1,
        .pollin         = 1,
        .buffer_select      = 1,
        .ioprio         = 1,
        .manual_alloc       = 1,
        .name           = "RECVMSG",
#if defined(CONFIG_NET)
        .async_size     = sizeof(struct io_async_msghdr),
        .prep           = io_recvmsg_prep,
        .issue          = io_recvmsg,
        .prep_async     = io_recvmsg_prep_async,
        .cleanup        = io_sendmsg_recvmsg_cleanup,
        .fail           = io_sendrecv_fail,
```

在第一次处理sqe的时候会强制使用IO_URING_F_NONBLOCK调用io_recvmsg

```C++
static inline void io_queue_sqe(struct io_kiocb *req)
    __must_hold(&req->ctx->uring_lock)
{
    int ret;

    ret = io_issue_sqe(req, IO_URING_F_NONBLOCK|IO_URING_F_COMPLETE_DEFER);

    /*
     * We async punt it if the file wasn't marked NOWAIT, or if the file
     * doesn't support non-blocking read/write attempts
     */
    if (likely(!ret))
        io_arm_ltimeout(req);
    else
        io_queue_async(req, ret);
}
```

对于recv的实现 如果传入了IO_URING_F_NONBLOCK默认就会走MSG_DONTWAIT放进flag里面，对于recvmsg系统调用这个flag会非阻塞尝试下看看有没有数据

所以第一次的非阻塞尝试与是不是block的**socket没关系**

```C++
int io_recvmsg(struct io_kiocb *req, unsigned int issue_flags)
{
   //....
    bool force_nonblock = issue_flags & IO_URING_F_NONBLOCK;
   //....
    flags = sr->msg_flags;
    if (force_nonblock)
        flags |= MSG_DONTWAIT;
        
    //....
    ret = __sys_recvmsg_sock(sock, &kmsg->msg, sr->umsg,
                     kmsg->uaddr, flags);
    //....                     
 }
```

若取不到数据直接走了io_queue_async -> io_arm_poll_handler -> __io_arm_poll_handler

```C++
if (def->pollin) {
//这里的recv op是支持 poll in的
//最后会直接挂载到vfs的poll上去
        mask |= EPOLLIN | EPOLLRDNORM;

        /* If reading from MSG_ERRQUEUE using recvmsg, ignore POLLIN */
        if (req->flags & REQ_F_CLEAR_POLLIN)
            mask &= ~EPOLLIN;
    } else {
        mask |= EPOLLOUT | EPOLLWRNORM;
    }
    if (def->poll_exclusive)
        mask |= EPOLLEXCLUSIVE;

    apoll = io_req_alloc_apoll(req, issue_flags);
    if (!apoll)
        return IO_APOLL_ABORTED;
    req->flags &= ~(REQ_F_SINGLE_POLL | REQ_F_DOUBLE_POLL);
    req->flags |= REQ_F_POLLED;
    ipt.pt._qproc = io_async_queue_proc;

    io_kbuf_recycle(req, issue_flags);
//简单来说就是 mask = vfs_poll(req->file, &ipt->pt) & poll->events;
    ret = __io_arm_poll_handler(req, &apoll->poll, &ipt, mask, issue_flags);
```