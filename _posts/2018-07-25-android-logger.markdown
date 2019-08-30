---
layout: post
title: "🍊Android Logger源码分析"
date: 2018-07-25
tags: android learn c
---
(Removed in Android Lollipop)

`简介`：轻量级的日志系统，以驱动的形式实现在内核中，提供了Java和C/C++接口。

`位置`：

`kernel/common/drivers/staging/android/logger.h`

`kernel/common/drivers/staging/android/logger.c`

## 数据结构

```clike
struct logger_entry {
	__u16		len;	/* length of the payload */
	__u16		__pad;	/* no matter what, we get 2 bytes of padding */
	__s32		pid;	/* generating process's pid */
	__s32		tid;	/* generating process's tid */
	__s32		sec;	/* seconds since Epoch */
	__s32		nsec;	/* nanoseconds */
	char		msg[0];	/* the entry's payload */
};
```

```clike
struct logger_log {
	unsigned char *		buffer;	/* the ring buffer itself */
	struct miscdevice	misc;	/* misc device representing the log */
	wait_queue_head_t	wq;	/* wait queue for readers */
	struct list_head	readers; /* this log's readers */
	struct mutex		mutex;	/* mutex protecting buffer */
	size_t			w_off;	/* current write head offset */
	size_t			head;	/* new readers start here */
	size_t			size;	/* size of the log */
};
```

```clike
struct logger_reader {
	struct logger_log *	log;	/* associated log */
	struct list_head	list;	/* entry in logger_log's list */
	size_t			r_off;	/* current read head offset */
};
```
## 详细描述

### logger_log

`logger_log`是最主要的数据结构

* `buffer`中保存着多条`logger_entry`

* `reader`中保存着正在读取日志的进程, 由`logger_reader`来描述

`buffer`作为环形缓冲区使用，因此给定一个`offerset`，需要取余来获得在`buffer`中的真实位置，logger中优化了取余操作:
```c
((n) & (log->size - 1))
```
当`log->size`为2^k时，该与运算结果即为取余。

### logger_entry

`logger_entry`是记录的最小单位，在`buffer`中，一条记录的形式为：`logger_entry|priority|tag|msg`。

注意，`logger_entry`最后`char msg[0]`中，`msg`指向的地址为`msg`自身所在的位置。

记录中除去`logger_entry`结构体外的长度，记录在`len`中。

### logger_reader

* `log`指向欲读取的缓冲区，由于Logger驱动程序会注册三个日志设备(类型: misc)，因此`reader`实际上有三种选择进行读取
* `list`包含`logger_log`中的list的条目，也就是用于连接其他`reader`

## Logger驱动的初始化过程

### static int __init logger_init(void);

```clike
static int __init logger_init(void)
{
	int ret;

	ret = init_log(&log_main);
	if (unlikely(ret))
		goto out;

	ret = init_log(&log_events);
	if (unlikely(ret))
		goto out;

	ret = init_log(&log_radio);
	if (unlikely(ret))
		goto out;

out:
	return ret;
}
```

* 初始化`log_main`
* 初始化`log_event`
* 初始化`log_radio`

### static int __init init_log(struct logger_log *log);

```clike
static int __init init_log(struct logger_log *log)
{
	int ret;

	ret = misc_register(&log->misc);
	if (unlikely(ret)) {
		printk(KERN_ERR "logger: failed to register misc "
		       "device for log '%s'!\n", log->misc.name);
		return ret;
	}

	printk(KERN_INFO "logger: created %luK log '%s'\n",
	       (unsigned long) log->size >> 10, log->misc.name);

	return 0;
}
```

简单的注册了一下设备`/dev/log/main` `/dev/log/event` `/dev/log/radio`。

## Logger驱动的日志读取过程

### static int logger_open(struct inode *inode, struct file *file);

```clike
static int logger_open(struct inode *inode, struct file *file)
{
	struct logger_log *log;
	int ret;

	ret = nonseekable_open(inode, file);
	if (ret)
		return ret;

	log = get_log_from_minor(MINOR(inode->i_rdev));
	if (!log)
		return -ENODEV;

	if (file->f_mode & FMODE_READ) {
		struct logger_reader *reader;

		reader = kmalloc(sizeof(struct logger_reader), GFP_KERNEL);
		if (!reader)
			return -ENOMEM;

		reader->log = log;
		INIT_LIST_HEAD(&reader->list);

		mutex_lock(&log->mutex);
		reader->r_off = log->head;
		list_add_tail(&reader->list, &log->readers);
		mutex_unlock(&log->mutex);

		file->private_data = reader;
	} else
		file->private_data = log;

	return 0;
}
```

设备在open时，如果是读设备，则`file->private_data = reader`，否则`file->private_data = log`。

于是在`logger_read`之前，创建好了`reader`并保存在了`file->private_data`中。

### static ssize_t logger_read(struct file *file, char __user *buf, size_t count, loff_t *pos);

```clike
static ssize_t logger_read(struct file *file, char __user *buf,
			   size_t count, loff_t *pos)
{
	struct logger_reader *reader = file->private_data;
	struct logger_log *log = reader->log;
	ssize_t ret;
	DEFINE_WAIT(wait);

start:
	while (1) {
		prepare_to_wait(&log->wq, &wait, TASK_INTERRUPTIBLE);

		mutex_lock(&log->mutex);
		ret = (log->w_off == reader->r_off);
		mutex_unlock(&log->mutex);
		if (!ret)
			break;

		if (file->f_flags & O_NONBLOCK) {
			ret = -EAGAIN;
			break;
		}

		if (signal_pending(current)) {
			ret = -EINTR;
			break;
		}

		schedule();
	}

	finish_wait(&log->wq, &wait);
	if (ret)
		return ret;

	mutex_lock(&log->mutex);

	/* is there still something to read or did we race? */
	if (unlikely(log->w_off == reader->r_off)) {
		mutex_unlock(&log->mutex);
		goto start;
	}

	/* get the size of the next entry */
	ret = get_entry_len(log, reader->r_off);
	if (count < ret) {
		ret = -EINVAL;
		goto out;
	}

	/* get exactly one entry from the log */
	ret = do_read_log_to_user(log, reader, buf, ret);

out:
	mutex_unlock(&log->mutex);

	return ret;
}
```

取出`file->private_data`中的`reader`，获取相关信息。在while循环中等待日志可读(log->w_off == reader->r_off，相等时不可读)，如果可读则跳出循环，进行读操作。

每次读取一条，首先需要读取`struct logger_entry`开头的`len`变量，其长度为两个字节，显然其地址必然指向`len`的第一个字节，但由于其保存在循环缓冲区中，第二个字节可能不是地址+1，还有可能是`buffer`的头部，因此在`get_entry_len()`中对此情况进行了判断。

真正的读取操作就比较简单了，使用`copy_to_user()`将数据拷贝到用户空间即可。

## Logger驱动的日志写入过程

从`logger_open()`知，当设备打开时为写模式时，`file->private_data`即为`log`，使用方法`logger_aio_write()`对设备进行写入。

### ssize_t logger_aio_write(struct kiocb *iocb, const struct iovec *iov, unsigned long nr_segs, loff_t ppos);

```clike
ssize_t logger_aio_write(struct kiocb *iocb, const struct iovec *iov,
			 unsigned long nr_segs, loff_t ppos)
{
	struct logger_log *log = file_get_log(iocb->ki_filp);
	size_t orig = log->w_off;
	struct logger_entry header;
	struct timespec now;
	ssize_t ret = 0;

	now = current_kernel_time();

	header.pid = current->tgid;
	header.tid = current->pid;
	header.sec = now.tv_sec;
	header.nsec = now.tv_nsec;
	header.len = min_t(size_t, iocb->ki_left, LOGGER_ENTRY_MAX_PAYLOAD);

	/* null writes succeed, return zero */
	if (unlikely(!header.len))
		return 0;

	mutex_lock(&log->mutex);

	/*
	 * Fix up any readers, pulling them forward to the first readable
	 * entry after (what will be) the new write offset. We do this now
	 * because if we partially fail, we can end up with clobbered log
	 * entries that encroach on readable buffer.
	 */
	fix_up_readers(log, sizeof(struct logger_entry) + header.len);

	do_write_log(log, &header, sizeof(struct logger_entry));

	while (nr_segs-- > 0) {
		size_t len;
		ssize_t nr;

		/* figure out how much of this vector we can keep */
		len = min_t(size_t, iov->iov_len, header.len - ret);

		/* write out this segment's payload */
		nr = do_write_log_from_user(log, iov->iov_base, len);
		if (unlikely(nr < 0)) {
			log->w_off = orig;
			mutex_unlock(&log->mutex);
			return nr;
		}

		iov++;
		ret += nr;
	}

	mutex_unlock(&log->mutex);

	/* wake up any blocked readers */
	wake_up_interruptible(&log->wq);

	return ret;
}
```

* `iocb`-io上下文
* `iov`-欲写入的内容
* `nr_segs`-长度（单位为段）
* `ppos`-没有用到

首先初始化一条entry，名为`header`，然后用`fix_up_readers()`调整`reader`的r_offset，由于`reader`读取的单位是`entry`，在循环缓冲区中写入记录，很可能会覆盖破坏掉某`reader`正在读取的某条记录，因此需要检测是否有reader在读取覆盖范围内的记录，进行修正，跳过这条记录，修正到下一条。

写入完记录后，`wake_up_interruptible()`来唤醒`reader`进程。

> logger驱动模块在退出系统时不会卸载