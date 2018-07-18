# lowmemorykiller原理分析

## 概述

Android的设计理念之一，即便是应用程序退出，但进程还会继续存在系统以便再次启动时提高响应时间。 这样的设计会带来一个问题, 每个进程都有自己独立的内存地址空间，随着应用打开数量的增多,系统已使用的内存越来越大，就很有可能导致系统内存不足, 那么需要一个能管理所有进程，根据一定策略来释放进程的策略，这便有了lmk，全称为LowMemoryKiller(低内存杀手)，由LowMemoryKiller来决定什么时间杀掉什么进程。

### 策略

1. 什么时候触发LowMemoryKiller

2. 触发LowMemoryKiller该回收哪个进程

   

Android制定了一套规则，用来评估内存水平，确定进程回收优先级，在不同内存水平下回收不同优先级进程

这套规则包含：

1. 给每个进程设置优先级adj
2. 设定可用内存与进程优先级关系，不同内存情况下，杀掉不同adj级别的进程

### 总体流程

![lowmemorykiller.png](/images/memory/lowmemorykiller.png)

1. AMS根据应用进程运行状态调用ProcessList类中的接口更新adj
2. ProcessList通过socket和守护进程lmkd进行通信，将更新的数据传给lmkd
3. lmkd将更新后的数据写入对应节点
4. 内存不足时kswapd线程会遍历shrinker链表，回调已注册的shrinker函数
5. shrinker回调LowMemoryKiller的函数，根据当前内存水平，找出需要杀掉的应用杀掉

### Adj

定义在ProcessList.java文件，oom_adj划分为16级，从-1000到1001之间取值。**adj值越大，优先级越低**，adj<0的进程都是系统进程。

| ADJ级别   | 取值|解释|
| -------- | -----  | -----  |
|UNKNOWN_ADJ|1001|一般指将要会缓存进程，无法获取确定值|
|CACHED_APP_MAX_ADJ|906|不可见进程的adj最大值|
|CACHED_APP_MIN_ADJ|900|不可见进程的adj最小值|
|SERVICE_B_AD| 800| B List中的Service（较老的、使用可能性更小）|
|PREVIOUS_APP_ADJ| 700|上一个App的进程(往往通过按返回键)|
|HOME_APP_ADJ | 600|Home进程|
|SERVICE_ADJ | 500|服务进程(**Service process**)|
|HEAVY_WEIGHT_APP_ADJ | 400|后台的重量级进程，system/rootdir/init.rc文件中设置|
|BACKUP_APP_ADJ | 300|备份进程|
|PERCEPTIBLE_APP_ADJ | 200|可感知进程，比如后台音乐播放  |
|VISIBLE_APP_ADJ | 100|可见进程(**Visible process**)|
|FOREGROUND_APP_ADJ | 0|前台进程（**Foreground process**)|
|PERSISTENT_SERVICE_ADJ | -700|关联着系统或persistent进程|
|PERSISTENT_PROC_ADJ | -800|系统persistent进程，比如telephony|
|SYSTEM_ADJ |-900|系统进程|
|NATIVE_ADJ | -1000|native进程（不被系统管理）|

lmkd会根据会根据当前系统可能内存的情况，来决定杀掉不同adj级别的进程。

### shrinker

LMK驱动通过注册shrinker来实现的，shrinker是linux kernel标准的回收内存page的机制，由内核线程kswapd负责监控。

当内存不足时kswapd线程会遍历一张shrinker链表，并回调已注册的shrinker函数来回收内存page，kswapd还会周期性唤醒来执行内存操作。每个zone维护active_list和inactive_list链表，内核根据页面活动状态将page在这两个链表之间移动，最终通过shrink_slab和shrink_zone来回收内存页。

## 代码实现

流程大致分为三层：

frameworks层： AMS、ProcessList

native层： lmkd

 kernel层：lowmemorykiller

### 核心方法

#### AMS

- 当AMS.applyOomAdjLocked()过程,则会设置某个进程的adj;
- 当AMS.updateConfiguration()过程中便会更新整个各个级别的oom_adj信息.
- 当AMS.cleanUpApplicationRecordLocked()或者handleAppDiedLocked()过程,则会将某个进程从lmkd策略中移除.

#### ProcessList

|方法|功能|命令|
|---|---|---|
|PL.setOomAdj()|设置进程adj|LMK_PROCPRIO|
|PL.updateOomLevels()|更新oom_adj|LMK_TARGET|
|PL.remove()|移除进程|LMK_PROCREMOVE|

ProcessList与lmkd通信定义了3种命令类型，这些文件的定义必须跟lmkd.c定义完全一致，格式分别如下：

    LMK_TARGET <minfree> <minkillprio> ... (up to 6 pairs)
    LMK_PROCPRIO <pid> <prio>
    LMK_PROCREMOVE <pid>

### frameworks层

#### 更新oom_adj

//frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java

``` java
    public void updateConfiguration(Configuration values) {
        enforceCallingPermission(android.Manifest.permission.CHANGE_CONFIGURATION,
                "updateConfiguration()");

        synchronized(this) {
            if (values == null && mWindowManager != null) {
                // sentinel: fetch the current configuration from the window manager
                values = mWindowManager.computeNewConfiguration();
            }

            if (mWindowManager != null) {
                mProcessList.applyDisplaySize(mWindowManager);
            }

            final long origId = Binder.clearCallingIdentity();
            if (values != null) {
                Settings.System.clearConfiguration(values);
            }
            updateConfigurationLocked(values, null, false);
            Binder.restoreCallingIdentity(origId);
        }
    }
```

// frameworks/base/services/core/java/com/android/server/am/ProcessList.java

```java
    void applyDisplaySize(WindowManagerService wm) {
        if (!mHaveDisplaySize) {
            Point p = new Point();
            wm.getBaseDisplaySize(Display.DEFAULT_DISPLAY, p);
            if (p.x != 0 && p.y != 0) {
                updateOomLevels(p.x, p.y, true);
                mHaveDisplaySize = true;
            }
        }
    }

    private void updateOomLevels(int displayWidth, int displayHeight, boolean write) {
        // Scale buckets from avail memory: at 300MB we use the lowest values to
        // 700MB or more for the top values.
        float scaleMem = ((float)(mTotalMemMb-350))/(700-350);

        // Scale buckets from screen size.
        int minSize = 480*800;  //  384000
        int maxSize = 1280*800; // 1024000  230400 870400  .264
        float scaleDisp = ((float)(displayWidth*displayHeight)-minSize)/(maxSize-minSize);
        if (true) {
            Slog.i("XXXXXX", "scaleMem=" + scaleMem);
            Slog.i("XXXXXX", "scaleDisp=" + scaleDisp + " dw=" + displayWidth
                    + " dh=" + displayHeight);
        }

        float scale = scaleMem > scaleDisp ? scaleMem : scaleDisp;
        if (scale < 0) scale = 0;
        else if (scale > 1) scale = 1;
        int minfree_adj = Resources.getSystem().getInteger(
                com.android.internal.R.integer.config_lowMemoryKillerMinFreeKbytesAdjust);
        int minfree_abs = Resources.getSystem().getInteger(
                com.android.internal.R.integer.config_lowMemoryKillerMinFreeKbytesAbsolute);
        if (true) {
            Slog.i("XXXXXX", "minfree_adj=" + minfree_adj + " minfree_abs=" + minfree_abs);
        }

        final boolean is64bit = Build.SUPPORTED_64_BIT_ABIS.length > 0;

        for (int i=0; i<mOomAdj.length; i++) {
            int low = mOomMinFreeLow[i];
            int high = mOomMinFreeHigh[i];
            if (is64bit) {
                // Increase the high min-free levels for cached processes for 64-bit
                if (i == 4) high = (high*3)/2;
                else if (i == 5) high = (high*7)/4;
            }
            mOomMinFree[i] = (int)(low + ((high-low)*scale));
        }

        if (minfree_abs >= 0) {
            for (int i=0; i<mOomAdj.length; i++) {
                mOomMinFree[i] = (int)((float)minfree_abs * mOomMinFree[i]
                        / mOomMinFree[mOomAdj.length - 1]);
            }
        }

        if (minfree_adj != 0) {
            for (int i=0; i<mOomAdj.length; i++) {
                mOomMinFree[i] += (int)((float)minfree_adj * mOomMinFree[i]
                        / mOomMinFree[mOomAdj.length - 1]);
                if (mOomMinFree[i] < 0) {
                    mOomMinFree[i] = 0;
                }
            }
        }

        // The maximum size we will restore a process from cached to background, when under
        // memory duress, is 1/3 the size we have reserved for kernel caches and other overhead
        // before killing background processes.
        mCachedRestoreLevel = (getMemLevel(ProcessList.CACHED_APP_MAX_ADJ)/1024) / 3;

        // Ask the kernel to try to keep enough memory free to allocate 3 full
        // screen 32bpp buffers without entering direct reclaim.
        int reserve = displayWidth * displayHeight * 4 * 3 / 1024;
        int reserve_adj = Resources.getSystem().getInteger(com.android.internal.R.integer.config_extraFreeKbytesAdjust);
        int reserve_abs = Resources.getSystem().getInteger(com.android.internal.R.integer.config_extraFreeKbytesAbsolute);

        if (reserve_abs >= 0) {
            reserve = reserve_abs;
        }

        if (reserve_adj != 0) {
            reserve += reserve_adj;
            if (reserve < 0) {
                reserve = 0;
            }
        }

        if (write) {
            ByteBuffer buf = ByteBuffer.allocate(4 * (2*mOomAdj.length + 1));
            buf.putInt(LMK_TARGET);
            for (int i=0; i<mOomAdj.length; i++) {
                buf.putInt((mOomMinFree[i]*1024)/PAGE_SIZE);
                buf.putInt(mOomAdj[i]);
            }

            writeLmkd(buf);
            SystemProperties.set("sys.sysctl.extra_free_kbytes", Integer.toString(reserve));
        }
        // GB: 2048,3072,4096,6144,7168,8192
        // HC: 8192,10240,12288,14336,16384,20480
    }

```

// frameworks/base/services/core/java/com/android/server/am/ProcessList.java

```java
    private static void writeLmkd(ByteBuffer buf) {

        for (int i = 0; i < 3; i++) {
            if (sLmkdSocket == null) {
                    if (openLmkdSocket() == false) {
                        try {
                            Thread.sleep(1000);
                        } catch (InterruptedException ie) {
                        }
                        continue;
                    }
            }

            try {
                sLmkdOutputStream.write(buf.array(), 0, buf.position());
                return;
            } catch (IOException ex) {
                Slog.w(TAG, "Error writing to lowmemorykiller socket");

                try {
                    sLmkdSocket.close();
                } catch (IOException ex2) {
                }

                sLmkdSocket = null;
            }
        }
    }
```

#### 更新应用adj

//frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java

```java
    private final boolean applyOomAdjLocked(ProcessRecord app, boolean doingAll, long now,
            long nowElapsed) {
       .....

        if (app.curAdj != app.setAdj) {
            ProcessList.setOomAdj(app.pid, app.info.uid, app.curAdj);
            app.setAdj = app.curAdj;
            app.verifiedAdj = ProcessList.INVALID_ADJ;
        }
        ....
    }
```

// frameworks/base/services/core/java/com/android/server/am/ProcessList.java

```java
    public static final void setOomAdj(int pid, int uid, int amt) {
        if (amt == UNKNOWN_ADJ)
            return;

        long start = SystemClock.elapsedRealtime();
        ByteBuffer buf = ByteBuffer.allocate(4 * 4);
        buf.putInt(LMK_PROCPRIO);
        buf.putInt(pid);
        buf.putInt(uid);
        buf.putInt(amt);
        writeLmkd(buf);
        long now = SystemClock.elapsedRealtime();
        if ((now-start) > 250) {
            Slog.w("ActivityManager", "SLOW OOM ADJ: " + (now-start) + "ms for pid " + pid
                    + " = " + amt);
        }
    }
```

### LMKD

#### 启动

lmkd是由init进程，通过解析lmkd.rc文件来启动的lmkd守护进程，lmkd会创建名为lmkd的socket，节点位于/dev/socket/lmkd，该socket用于跟上层framework交互

lmkd.rc源文件位于system/core/lmkd/lmkd.rc，在mk文件中加入LOCAL_INIT_RC := lmkd.rc，编译后路径为 /system/etc/init/lmkd.rc

```shell
service lmkd /system/bin/lmkd
    class core
    group root readproc
    critical
    socket lmkd seqpacket 0660 system system
    writepid /dev/cpuset/system-background/tasks
```

#### main

// system/core/lmkd/lmkd.c

```c
int main(int argc __unused, char **argv __unused) {
    struct sched_param param = {
            .sched_priority = 1,
    };

    mlockall(MCL_FUTURE);
    sched_setscheduler(0, SCHED_FIFO, &param);
    if (!init())
        mainloop();

    ALOGI("exiting");
    return 0;
}

```

#### init

```c
static int init(void) {
    struct epoll_event epev;
    int i;
    int ret;

    page_k = sysconf(_SC_PAGESIZE);
    if (page_k == -1)
        page_k = PAGE_SIZE;
    page_k /= 1024;

    epollfd = epoll_create(MAX_EPOLL_EVENTS);
    if (epollfd == -1) {
        ALOGE("epoll_create failed (errno=%d)", errno);
        return -1;
    }

    ctrl_lfd = android_get_control_socket("lmkd");
    if (ctrl_lfd < 0) {
        ALOGE("get lmkd control socket failed");
        return -1;
    }

    ret = listen(ctrl_lfd, 1);
    if (ret < 0) {
        ALOGE("lmkd control socket listen failed (errno=%d)", errno);
        return -1;
    }

    epev.events = EPOLLIN;
    epev.data.ptr = (void *)ctrl_connect_handler;
    if (epoll_ctl(epollfd, EPOLL_CTL_ADD, ctrl_lfd, &epev) == -1) {
        ALOGE("epoll_ctl for lmkd control socket failed (errno=%d)", errno);
        return -1;
    }
    maxevents++;

    use_inkernel_interface = !access(INKERNEL_MINFREE_PATH, W_OK);

    if (use_inkernel_interface) {
        ALOGI("Using in-kernel low memory killer interface");
    } else {
        ret = init_mp(MEMPRESSURE_WATCH_LEVEL, (void *)&mp_event);
        if (ret)
            ALOGE("Kernel does not support memory pressure events or in-kernel low memory killer");
    }

    for (i = 0; i <= ADJTOSLOT(OOM_SCORE_ADJ_MAX); i++) {
        procadjslot_list[i].next = &procadjslot_list[i];
        procadjslot_list[i].prev = &procadjslot_list[i];
    }

    return 0;
}
```

epoll_create创建一个epoll的句柄，最大监听3个事件。

使用epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);注册事件。
第一个参数是epoll_create()的返回值，
第二个参数表示动作，用三个宏来表示：
EPOLL_CTL_ADD：注册新的fd到epfd中；
EPOLL_CTL_MOD：修改已经注册的fd的监听事件；
EPOLL_CTL_DEL：从epfd中删除一个fd；
第三个参数是需要监听的fd.
第四个参数是告诉内核需要监听什么事

#### mainloop

```c
static void mainloop(void) {
    while (1) {
        struct epoll_event events[maxevents];
        int nevents;
        int i;

        ctrl_dfd_reopened = 0;
        nevents = epoll_wait(epollfd, events, maxevents, -1);

        if (nevents == -1) {
            if (errno == EINTR)
                continue;
            ALOGE("epoll_wait failed (errno=%d)", errno);
            continue;
        }

        for (i = 0; i < nevents; ++i) {
            if (events[i].events & EPOLLERR)
                ALOGD("EPOLLERR on event #%d", i);
            if (events[i].data.ptr)
                (*(void (*)(uint32_t))events[i].data.ptr)(events[i].events);
        }
    }
}
```

等待事件发生，当事件发生调用事件处理函数。这里有对应事件传进来会先调用ctrl_connect_handler函数。

#### ctrl_connect_handler

```c
static void ctrl_connect_handler(uint32_t events __unused) {
    struct sockaddr_storage ss;
    struct sockaddr *addrp = (struct sockaddr *)&ss;
    socklen_t alen;
    struct epoll_event epev;

    if (ctrl_dfd >= 0) {
        ctrl_data_close();
        ctrl_dfd_reopened = 1;
    }

    alen = sizeof(ss);
    ctrl_dfd = accept(ctrl_lfd, addrp, &alen);

    if (ctrl_dfd < 0) {
        ALOGE("lmkd control socket accept failed; errno=%d", errno);
        return;
    }

    ALOGI("ActivityManager connected");
    maxevents++;
    epev.events = EPOLLIN;
    epev.data.ptr = (void *)ctrl_data_handler;
    if (epoll_ctl(epollfd, EPOLL_CTL_ADD, ctrl_dfd, &epev) == -1) {
        ALOGE("epoll_ctl for data connection socket failed; errno=%d", errno);
        ctrl_data_close();
        return;
    }
}
```

主要是监听ctrl_dfd， 当EPOLLIN事件发生时，ctrl_data_handler会被调用。

#### ctrl_data_handler

```c
static void ctrl_data_handler(uint32_t events) {
    if (events & EPOLLHUP) {
        ALOGI("ActivityManager disconnected");
        if (!ctrl_dfd_reopened)
            ctrl_data_close();
    } else if (events & EPOLLIN) {
        ctrl_command_handler();
    }
}

static void ctrl_command_handler(void) {
    int ibuf[CTRL_PACKET_MAX / sizeof(int)];
    int len;
    int cmd = -1;
    int nargs;
    int targets;

    len = ctrl_data_read((char *)ibuf, CTRL_PACKET_MAX);
    if (len <= 0)
        return;

    nargs = len / sizeof(int) - 1;
    if (nargs < 0)
        goto wronglen;

    cmd = ntohl(ibuf[0]);

    switch(cmd) {
    case LMK_TARGET:
        targets = nargs / 2;
        if (nargs & 0x1 || targets > (int)ARRAY_SIZE(lowmem_adj))
            goto wronglen;
        cmd_target(targets, &ibuf[1]);
        break;
    case LMK_PROCPRIO:
        if (nargs != 3)
            goto wronglen;
        cmd_procprio(ntohl(ibuf[1]), ntohl(ibuf[2]), ntohl(ibuf[3]));
        break;
    case LMK_PROCREMOVE:
        if (nargs != 1)
            goto wronglen;
        cmd_procremove(ntohl(ibuf[1]));
        break;
    default:
        ALOGE("Received unknown command code %d", cmd);
        return;
    }

    return;

wronglen:
    ALOGE("Wrong control socket read length cmd=%d len=%d", cmd, len);
}
```

解析socket传来的数据：

- 当消息类型为LMK_TARGET时，调用cmd_target， 分别向/sys/module/lowmemorykiller/parameters/minfree、/sys/module/lowmemorykiller/parameters/adj写入评估标准数据
- 当消息类型为LMK_PROCPRIO，调用cmd_procprio，更新对应进程的adj值，即更新/proc/pid/oom_score_adj值

#### cmd_target

```c
static void cmd_target(int ntargets, int *params) {
    int i;

    if (ntargets > (int)ARRAY_SIZE(lowmem_adj))
        return;

    for (i = 0; i < ntargets; i++) {
        lowmem_minfree[i] = ntohl(*params++);
        lowmem_adj[i] = ntohl(*params++);
    }

    lowmem_targets_size = ntargets;

    if (use_inkernel_interface) {
        char minfreestr[128];
        char killpriostr[128];

        minfreestr[0] = '\0';
        killpriostr[0] = '\0';

        for (i = 0; i < lowmem_targets_size; i++) {
            char val[40];

            if (i) {
                strlcat(minfreestr, ",", sizeof(minfreestr));
                strlcat(killpriostr, ",", sizeof(killpriostr));
            }

            snprintf(val, sizeof(val), "%d", lowmem_minfree[i]);
            strlcat(minfreestr, val, sizeof(minfreestr));
            snprintf(val, sizeof(val), "%d", lowmem_adj[i]);
            strlcat(killpriostr, val, sizeof(killpriostr));
        }

        writefilestring(INKERNEL_MINFREE_PATH, minfreestr);
        writefilestring(INKERNEL_ADJ_PATH, killpriostr);
    }
}
```

#### cmd_procprio

```c
static void cmd_procprio(int pid, int uid, int oomadj) {
    struct proc *procp;
    char path[80];
    char val[20];

    if (oomadj < OOM_SCORE_ADJ_MIN || oomadj > OOM_SCORE_ADJ_MAX) {
        ALOGE("Invalid PROCPRIO oomadj argument %d", oomadj);
        return;
    }

    snprintf(path, sizeof(path), "/proc/%d/oom_score_adj", pid);
    snprintf(val, sizeof(val), "%d", oomadj);
    writefilestring(path, val);

    if (use_inkernel_interface)
        return;

    procp = pid_lookup(pid);
    if (!procp) {
            procp = malloc(sizeof(struct proc));
            if (!procp) {
                // Oh, the irony.  May need to rebuild our state.
                return;
            }

            procp->pid = pid;
            procp->uid = uid;
            procp->oomadj = oomadj;
            proc_insert(procp);
    } else {
        proc_unslot(procp);
        procp->oomadj = oomadj;
        proc_slot(procp);
    }
}
```

### kernel层

#### lowmemorykiller初始化

lowmemorykiller driver位于 drivers/staging/Android/lowmemorykiller.c

```c
module_init(lowmem_init);
module_exit(lowmem_exit);

static int __init lowmem_init(void)
{
    register_shrinker(&lowmem_shrinker);
    return 0;
}

static void __exit lowmem_exit(void)
{
    unregister_shrinker(&lowmem_shrinker);
}

```

#### lowmem_shrink

```c
static int lowmem_shrink(struct shrinker *s, struct shrink_control *sc)
{
    struct task_struct *tsk;
    struct task_struct *selected = NULL;
    int rem = 0;
    int tasksize;
    int i;
    short min_score_adj = OOM_SCORE_ADJ_MAX + 1;
    int minfree = 0;
    int selected_tasksize = 0;
    short selected_oom_score_adj;
    int array_size = ARRAY_SIZE(lowmem_adj);
    int other_free = global_page_state(NR_FREE_PAGES) - totalreserve_pages;
    int other_file = global_page_state(NR_FILE_PAGES) -
                        global_page_state(NR_SHMEM);

    if (lowmem_adj_size < array_size)
        array_size = lowmem_adj_size;
    if (lowmem_minfree_size < array_size)
        array_size = lowmem_minfree_size;
    for (i = 0; i < array_size; i++) {
        minfree = lowmem_minfree[i];
        if (other_free < minfree && other_file < minfree) {
            min_score_adj = lowmem_adj[i];
            break;
        }
    }
    if (sc->nr_to_scan > 0)
        lowmem_print(3, "lowmem_shrink %lu, %x, ofree %d %d, ma %hd\n",
                sc->nr_to_scan, sc->gfp_mask, other_free,
                other_file, min_score_adj);
    rem = global_page_state(NR_ACTIVE_ANON) +
        global_page_state(NR_ACTIVE_FILE) +
        global_page_state(NR_INACTIVE_ANON) +
        global_page_state(NR_INACTIVE_FILE);
    if (sc->nr_to_scan <= 0 || min_score_adj == OOM_SCORE_ADJ_MAX + 1) {
        lowmem_print(5, "lowmem_shrink %lu, %x, return %d\n",
                 sc->nr_to_scan, sc->gfp_mask, rem);
        return rem;
    }
    selected_oom_score_adj = min_score_adj;

    rcu_read_lock();
    for_each_process(tsk) {
        struct task_struct *p;
        short oom_score_adj;

        if (tsk->flags & PF_KTHREAD)
            continue;

        p = find_lock_task_mm(tsk);
        if (!p)
            continue;

        if (test_tsk_thread_flag(p, TIF_MEMDIE) &&
            time_before_eq(jiffies, lowmem_deathpending_timeout)) {
            task_unlock(p);
            rcu_read_unlock();
            return 0;
        }
        oom_score_adj = p->signal->oom_score_adj;
        if (oom_score_adj < min_score_adj) {
            task_unlock(p);
            continue;
        }
        tasksize = get_mm_rss(p->mm);
        task_unlock(p);
        if (tasksize <= 0)
            continue;
        if (selected) {
            if (oom_score_adj < selected_oom_score_adj)
                continue;
            if (oom_score_adj == selected_oom_score_adj &&
                tasksize <= selected_tasksize)
                continue;
        }
        selected = p;
        selected_tasksize = tasksize;
        selected_oom_score_adj = oom_score_adj;
        lowmem_print(2, "select '%s' (%d), adj %hd, size %d, to kill\n",
                 p->comm, p->pid, oom_score_adj, tasksize);
    }
    if (selected) {
        lowmem_print(1, "Killing '%s' (%d), adj %hd,\n" \
                "   to free %ldkB on behalf of '%s' (%d) because\n" \
                "   cache %ldkB is below limit %ldkB for oom_score_adj %hd\n" \
                "   Free memory is %ldkB above reserved\n",
                 selected->comm, selected->pid,
                 selected_oom_score_adj,
                 selected_tasksize * (long)(PAGE_SIZE / 1024),
                 current->comm, current->pid,
                 other_file * (long)(PAGE_SIZE / 1024),
                 minfree * (long)(PAGE_SIZE / 1024),
                 min_score_adj,
                 other_free * (long)(PAGE_SIZE / 1024));
        lowmem_deathpending_timeout = jiffies + HZ;
        send_sig(SIGKILL, selected, 0);
        set_tsk_thread_flag(selected, TIF_MEMDIE);
        rem -= selected_tasksize;
    }
    lowmem_print(4, "lowmem_shrink %lu, %x, return %d\n",
             sc->nr_to_scan, sc->gfp_mask, rem);
    rcu_read_unlock();
    return rem;
}

```

1. 获取当前剩余内存大小other_free、other_file
2. 根据当前内存other_free、other_file大小，获取当前能杀掉的最小的adj值min_score_adj。
3. 在所有adj>=min_score_adj的进程中，选择adj最大的进程杀掉，如果adj最大的进程有多个，则杀掉rss内存最大的进程

### 其他

哪些场景下都会触发`updateOomAdjLocked`来更新进程adj:

#### Activity

- ASS.realStartActivityLocked:  启动Activity
- AS.resumeTopActivityInnerLocked: 恢复栈顶Activity
- AS.finishCurrentActivityLocked: 结束当前Activity
- AS.destroyActivityLocked: 摧毁当前Activity

#### Service

位于ActiveServices.java

- realStartServiceLocked: 启动服务
- bindServiceLocked: 绑定服务(只更新当前app)
- unbindServiceLocked: 解绑服务 (只更新当前app)
- bringDownServiceLocked: 结束服务 (只更新当前app)
- sendServiceArgsLocked: 在bringup或则cleanup服务过程调用 (只更新当前app)

#### broadcast

- BQ.processNextBroadcast: 处理下一个广播
- BQ.processCurBroadcastLocked: 处理当前广播
- BQ.deliverToRegisteredReceiverLocked: 分发已注册的广播 (只更新当前app)

#### ContentProvider

- AMS.removeContentProvider: 移除provider
- AMS.publishContentProviders: 发布provider (只更新当前app)
- AMS.getContentProviderImpl: 获取provider (只更新当前app)

#### Process

位于ActivityManagerService.java

- setSystemProcess: 创建并设置系统进程
- addAppLocked: 创建persistent进程
- attachApplicationLocked: 进程创建后attach到system_server的过程;
- trimApplications: 清除没有使用app
- appDiedLocked: 进程死亡
- killAllBackgroundProcesses: 杀死所有后台进程.即(ADJ>9或removed=true的普通进程)
- killPackageProcessesLocked: 以包名的形式 杀掉相关进程;

## 总结

本文主要从frameworks的ProcessList.java调整adj，通过socket通信将事件发送给native的守护进程lmkd；lmkd再根据具体的命令来执行相应操作，其主要功能
更新进程的oom_score_adj值以及lowmemorykiller驱动的parameters(包括minfree和adj)；

最后讲到了lowmemorykiller驱动，通过注册shrinker，借助linux标准的内存回收机制，根据当前系统可用内存以及parameters配置参数(adj,minfree)来选取合适的selected_oom_score_adj，再从所有进程中选择adj大于该目标值的并且占用rss内存最大的进程，将其杀掉，从而释放出内存。