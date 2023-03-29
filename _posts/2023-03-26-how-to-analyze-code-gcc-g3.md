---
layout: post
title: "如何阅读代码 -- 巧借 GCC 选项分析真正使用的宏"
description: "如何阅读代码序列"
date: 2023-03-26
tags: [OpenResty, Nginx, Lua, LuaJIT, GC]
---

# 阅读代码的难题

开源软件一般都会为了兼容各种不同的操作系统、不同的依赖软件版本、调试等目的而定义很多的宏。

这些宏给代码阅读带来了很多的障碍，分不清楚到底周到哪个代码路径。
特别是分析Bug的时候，更希望确认具体走的是哪个具体的代码路径。

我们怎么才能够确认这些宏是否定义呢？宏定义的值是多少呢？

如果你想通过查看源码的方式来确认这些宏的定义，那么就形成了先有鸡还是先有蛋的问题。
要是有个工具可以准确的列出编译的软件使用的是哪些宏该有多幸福。而这样的工具确实存在，
那就是 GCC 本身。

# GCC -g3 选项让编译结果包含宏定义

编译软件加上 `-g3` 就可以实现我们的目的。而 `-g3` 是什么东西呢？

GCC [官方文档](https://gcc.gnu.org/onlinedocs/gcc/Debugging-Options.html) 是这么解释 `-g3` 选项的。


```
-glevel
-ggdblevel
-gvmslevel
  Request debugging information and also use level to specify how much information. The default level is 2.

  Level 0 produces no debug information at all. Thus, -g0 negates -g.

  Level 1 produces minimal information, enough for making backtraces in parts of the program that you don’t plan to debug. This includes descriptions of functions and external variables, and line number tables, but no information about local variables.

  Level 3 includes extra information, such as all the macro definitions present in the program. Some debuggers support macro expansion when you use -g3.

  If you use multiple -g options, with or without level numbers, the last such option is the one that is effective.
```

可以看到，加上 `-g3` 选项后会包含额外的信息，其中就有程序中使用的宏定义。

因此，我们只要加上 `-g3` 的编译选项，我们编译的程序的可执行文件就包含了宏定义的。也就是说我们现在拥有了一座宝藏，如何查看这些宝藏呢？

# objdump 查看程序具体的宏定义

我们一般都只是执行程序得到结果，未曾想到可执行程序竟然包含了大量的其它信息。
如何查询这些信息呢？这里就要用到 objdump 这个工具。

下面给出了一个例子

```sh
[ljl@localhost liblmdb]$ objdump --dwarf=macro liblmdb.so  | grep LOCK_MUTEX0
 DW_MACRO_define_strp - lineno : 442 macro : LOCK_MUTEX0(mutex) pthread_mutex_lock(mutex)
 DW_MACRO_define_strp - lineno : 499 macro : LOCK_MUTEX(rc,env,mutex) (((rc) = LOCK_MUTEX0(mutex)) && ((rc) = mdb_mutex_failed(env, mutex, rc)))
```

## 案例对应的部分代码

下面以 lmdb/libraries/liblmdb 文件的例子

```
#ifdef _WIN32
#define MDB_USE_HASH	1
#define MDB_PIDLOCK	0
#define THREAD_RET	DWORD
#define pthread_t	HANDLE
#define pthread_mutex_t	HANDLE
#define pthread_cond_t	HANDLE
typedef HANDLE mdb_mutex_t, mdb_mutexref_t;
#define pthread_key_t	DWORD
#define pthread_self()	GetCurrentThreadId()
#define pthread_key_create(x,y)	\
	((*(x) = TlsAlloc()) == TLS_OUT_OF_INDEXES ? ErrCode() : 0)
#define pthread_key_delete(x)	TlsFree(x)
#define pthread_getspecific(x)	TlsGetValue(x)
#define pthread_setspecific(x,y)	(TlsSetValue(x,y) ? 0 : ErrCode())
#define pthread_mutex_unlock(x)	ReleaseMutex(*x)
#define pthread_mutex_lock(x)	WaitForSingleObject(*x, INFINITE)
#define pthread_cond_signal(x)	SetEvent(*x)
#define pthread_cond_wait(cond,mutex)	do{SignalObjectAndWait(*mutex, *cond, INFINITE, FALSE); WaitForSingleObject(*mutex, INFINITE);}while(0)
#define THREAD_CREATE(thr,start,arg) \
	(((thr) = CreateThread(NULL, 0, start, arg, 0, NULL)) ? 0 : ErrCode())
#define THREAD_FINISH(thr) \
	(WaitForSingleObject(thr, INFINITE) ? ErrCode() : 0)
#define LOCK_MUTEX0(mutex)		WaitForSingleObject(mutex, INFINITE)
#define UNLOCK_MUTEX(mutex)		ReleaseMutex(mutex)
#define mdb_mutex_consistent(mutex)	0
#define getpid()	GetCurrentProcessId()
#define	MDB_FDATASYNC(fd)	(!FlushFileBuffers(fd))
#define	MDB_MSYNC(addr,len,flags)	(!FlushViewOfFile(addr,len))
#define	ErrCode()	GetLastError()
#define GET_PAGESIZE(x) {SYSTEM_INFO si; GetSystemInfo(&si); (x) = si.dwPageSize;}
#define	close(fd)	(CloseHandle(fd) ? 0 : -1)
#define	munmap(ptr,len)	UnmapViewOfFile(ptr)
#ifdef PROCESS_QUERY_LIMITED_INFORMATION
#define MDB_PROCESS_QUERY_LIMITED_INFORMATION PROCESS_QUERY_LIMITED_INFORMATION
#else
#define MDB_PROCESS_QUERY_LIMITED_INFORMATION 0x1000
#endif
#else
#define THREAD_RET	void *
#define THREAD_CREATE(thr,start,arg)	pthread_create(&thr,NULL,start,arg)
#define THREAD_FINISH(thr)	pthread_join(thr,NULL)

	/** For MDB_LOCK_FORMAT: True if readers take a pid lock in the lockfile */
#define MDB_PIDLOCK			1

#ifdef MDB_USE_POSIX_SEM

typedef sem_t *mdb_mutex_t, *mdb_mutexref_t;
#define LOCK_MUTEX0(mutex)		mdb_sem_wait(mutex)
#define UNLOCK_MUTEX(mutex)		sem_post(mutex)

static int
mdb_sem_wait(sem_t *sem)
{
   int rc;
   while ((rc = sem_wait(sem)) && (rc = errno) == EINTR) ;
   return rc;
}

#elif defined MDB_USE_SYSV_SEM

typedef struct mdb_mutex {
	int semid;
	int semnum;
	int *locked;
} mdb_mutex_t[1], *mdb_mutexref_t;

#define LOCK_MUTEX0(mutex)		mdb_sem_wait(mutex)
#define UNLOCK_MUTEX(mutex)		do { \
	struct sembuf sb = { 0, 1, SEM_UNDO }; \
	sb.sem_num = (mutex)->semnum; \
	*(mutex)->locked = 0; \
	semop((mutex)->semid, &sb, 1); \
} while(0)

static int
mdb_sem_wait(mdb_mutexref_t sem)
{
	int rc, *locked = sem->locked;
	struct sembuf sb = { 0, -1, SEM_UNDO };
	sb.sem_num = sem->semnum;
	do {
		if (!semop(sem->semid, &sb, 1)) {
			rc = *locked ? MDB_OWNERDEAD : MDB_SUCCESS;
			*locked = 1;
			break;
		}
	} while ((rc = errno) == EINTR);
	return rc;
}

#define mdb_mutex_consistent(mutex)	0

#else	/* MDB_USE_POSIX_MUTEX: */
	/** Shared mutex/semaphore as the original is stored.
	 *
	 *	Not for copies.  Instead it can be assigned to an #mdb_mutexref_t.
	 *	When mdb_mutexref_t is a pointer and mdb_mutex_t is not, then it
	 *	is array[size 1] so it can be assigned to the pointer.
	 */
typedef pthread_mutex_t mdb_mutex_t[1];
	/** Reference to an #mdb_mutex_t */
typedef pthread_mutex_t *mdb_mutexref_t;
	/** Lock the reader or writer mutex.
	 *	Returns 0 or a code to give #mdb_mutex_failed(), as in #LOCK_MUTEX().
	 */
#define LOCK_MUTEX0(mutex)	pthread_mutex_lock(mutex)
	/** Unlock the reader or writer mutex.
	 */
#define UNLOCK_MUTEX(mutex)	pthread_mutex_unlock(mutex)
	/** Mark mutex-protected data as repaired, after death of previous owner.
	 */
#define mdb_mutex_consistent(mutex)	pthread_mutex_consistent(mutex)
#endif	/* MDB_USE_POSIX_SEM || MDB_USE_SYSV_SEM */
#endif
```

