# init进程

> 基于代码4.4.2
> 相关代码：
> linux-3.10/init/main.c

## 简介

Linux下有3个特殊的进程，idle进程(PID = 0), init进程(PID = 1)和kthreadd(PID = 2)

init进程由idle通过kernel_thread创建。内核空间完成初始化后，会加载init程序并最终运运行在用户空间，进程号为1，init启动后会调用init中的main()方法，执行init进程的职责。

```c
static int __ref kernel_init(void *unused)
{
    kernel_init_freeable();
    /* need to finish all async __init code before freeing the memory */
    async_synchronize_full();
    free_initmem();
    mark_rodata_ro();
    system_state = SYSTEM_RUNNING;
    numa_default_policy();

    flush_delayed_fput();

    if (ramdisk_execute_command) {
        if (!run_init_process(ramdisk_execute_command))
            return 0;
        pr_err("Failed to execute %s\n", ramdisk_execute_command);
    }

    /*
     * We try each of these until one succeeds.
     *
     * The Bourne shell can be used instead of init if we are
     * trying to recover a really broken machine.
     */
    if (execute_command) {
        if (!run_init_process(execute_command))
            return 0;
        pr_err("Failed to execute %s.  Attempting defaults...\n",
            execute_command);
    }
    if (!run_init_process("/sbin/init") ||
        !run_init_process("/etc/init") ||
        !run_init_process("/bin/init") ||
        !run_init_process("/bin/sh"))
        return 0;

    panic("No init found.  Try passing init= option to kernel. "
          "See Linux Documentation/init.txt for guidance.");
}
```

内核挂接根文件系统成功以后，将通过run_init_process()函数执行应用程序，如果ramdisk_execute_command存在，则执行ramdisk_execute_command；如果execute_command存在，则执行execute_command；如果不存在，则顺序执行/sbin/init、/etc/init、/bin/init、/bin/sh，直到有一个执行成功为止。如果都不存在，则会调用panic()上报错误。

init启动后将会调用init.main函数

### main

```c
int main(int argc, char **argv)
{
    int fd_count = 0;
    struct pollfd ufds[4];
    char *tmpdev;
    char* debuggable;
    char tmp[32];
    int property_set_fd_init = 0;
    int signal_fd_init = 0;
    int keychord_fd_init = 0;
    bool is_charger = false;

    char* args_swapon[2];
    args_swapon[0] = "swapon_all";;
    args_swapon[1] = "/fstab.sun8iw11p1";;

    char* args_write[3];
    args_write[0] = "write";
    args_write[1] = "/proc/sys/vm/page-cluster";
    args_write[2] = "0";

    if (!strcmp(basename(argv[0]), "ueventd"))
        return ueventd_main(argc, argv);

    /*if (!strcmp(basename(argv[0]), "watchdogd"))
        return watchdogd_main(argc, argv);

    if (!strcmp(basename(argv[0]), "fswatcherd"))
        return fswatcherd_main(argc, argv);*/

    /* clear the umask */
    ERROR("init proc start\n");
    umask(0);

        /* Get the basic filesystem setup we need put
         * together in the initramdisk on / and then we'll
         * let the rc file figure out the rest.
         */
    mkdir("/dev", 0755);
    mkdir("/proc", 0755);
    mkdir("/sys", 0755);

    mount("tmpfs", "/dev", "tmpfs", MS_NOSUID, "mode=0755");
    mkdir("/dev/pts", 0755);
    mkdir("/dev/socket", 0755);
    mount("devpts", "/dev/pts", "devpts", 0, NULL);
    mount("proc", "/proc", "proc", 0, NULL);
    mount("sysfs", "/sys", "sysfs", 0, NULL);

        /* indicate that booting is in progress to background fw loaders, etc */
    close(open("/dev/.booting", O_WRONLY | O_CREAT, 0000));

        /* We must have some place other than / to create the
         * device nodes for kmsg and null, otherwise we won't
         * be able to remount / read-only later on.
         * Now that tmpfs is mounted on /dev, we can actually
         * talk to the outside world.
         */
    open_devnull_stdio();
    klog_init();
    property_init();

    get_hardware_name(hardware, &revision);

    process_kernel_cmdline();

#if ASYNC_INIT_SELINUX
	int err = 0;
	int selinux_tid = 0;
    INFO("Selinux async initialize...\n");
	err = pthread_create(&selinux_tid, NULL, selinux_init_thread, NULL);
	if (err != 0)
	{
		ERROR("create selinux init thread failed: %s\n", strerror(err));
		return -1;
	}
#else
    INFO("Selinux sync initialize...\n");
    union selinux_callback cb;
    cb.func_log = klog_write;
    selinux_set_callback(SELINUX_CB_LOG, cb);

    cb.func_audit = audit_callback;
    selinux_set_callback(SELINUX_CB_AUDIT, cb);

    selinux_initialize();
    /* These directories were necessarily created before initial policy load
     * and therefore need their security context restored to the proper value.
     * This must happen before /dev is populated by ueventd.
     */
#endif

    if (!selinux_is_disabled()) {
        restorecon("/dev");
        restorecon("/dev/socket");
        restorecon("/dev/__properties__");
        restorecon_recursive("/sys");
    }

    is_charger = !strcmp(bootmode, "charger");

    INFO("property init\n");
    if (!is_charger)
        property_load_boot_defaults();
	get_kernel_cmdline_partitions();
    get_kernel_cmdline_signature();
    INFO("reading config file\n");
    init_parse_config_file("/init.rc");

    action_for_each_trigger("early-init", action_add_queue_tail);

    /* execute all the boot actions to get us started */
    action_for_each_trigger("init", action_add_queue_tail);

    /* skip mounting filesystems in charger mode */
    if (!is_charger) {
        action_for_each_trigger("early-fs", action_add_queue_tail);
        queue_builtin_action(wait_for_coldboot_done_action, "wait_for_coldboot_done");
        queue_builtin_action(console_init_action, "console_init");
        action_for_each_trigger("fs", action_add_queue_tail);
        action_for_each_trigger("post-fs", action_add_queue_tail);
        action_for_each_trigger("post-fs-data", action_add_queue_tail);
        //SWAP TO ZRAM if low mem devices
       // if (!(get_dram_size() > 512)) {
            char trigger[] = {"early-fs"};
            ERROR("***************************LOW MEM DEVICE DETECT");
            add_command(trigger, 2, args_swapon);
            char trigger2[] = {"post-fs-data"};
            add_command(trigger2, 3, args_write);
        //}
    }

    queue_builtin_action(property_service_init_action, "property_service_init");
    queue_builtin_action(init_set_disp_policy, "init_set_disp_policy");
    queue_builtin_action(signal_init_action, "signal_init");
    queue_builtin_action(check_startup_action, "check_startup");

    if (is_charger) {
        action_for_each_trigger("charger", action_add_queue_tail);
    } else {
        action_for_each_trigger("early-boot", action_add_queue_tail);
        action_for_each_trigger("boot", action_add_queue_tail);
    }

        /* run all property triggers based on current state of the properties */
    queue_builtin_action(queue_property_triggers_action, "queue_property_triggers");


#if BOOTCHART
    queue_builtin_action(bootchart_init_action, "bootchart_init");
#endif

    for(;;) {
        int nr, i, timeout = -1;

        execute_one_command();
        restart_processes();

        if (!property_set_fd_init && get_property_set_fd() > 0) {
            ufds[fd_count].fd = get_property_set_fd();
            ufds[fd_count].events = POLLIN;
            ufds[fd_count].revents = 0;
            fd_count++;
            property_set_fd_init = 1;
        }
        if (!signal_fd_init && get_signal_fd() > 0) {
            ufds[fd_count].fd = get_signal_fd();
            ufds[fd_count].events = POLLIN;
            ufds[fd_count].revents = 0;
            fd_count++;
            signal_fd_init = 1;
        }
        if (!keychord_fd_init && get_keychord_fd() > 0) {
            ufds[fd_count].fd = get_keychord_fd();
            ufds[fd_count].events = POLLIN;
            ufds[fd_count].revents = 0;
            fd_count++;
            keychord_fd_init = 1;
        }

        if (process_needs_restart) {
            timeout = (process_needs_restart - gettime()) * 1000;
            if (timeout < 0)
                timeout = 0;
        }

        if (!action_queue_empty() || cur_action)
            timeout = 0;

#if BOOTCHART
        if (bootchart_count > 0) {
            if (timeout < 0 || timeout > BOOTCHART_POLLING_MS)
                timeout = BOOTCHART_POLLING_MS;
            if (bootchart_step() < 0 || --bootchart_count == 0) {
                bootchart_finish();
                bootchart_count = 0;
            }
        }
#endif

        nr = poll(ufds, fd_count, timeout);
        if (nr <= 0)
            continue;

        for (i = 0; i < fd_count; i++) {
            if (ufds[i].revents == POLLIN) {
                if (ufds[i].fd == get_property_set_fd())
                    handle_property_set_fd();
                else if (ufds[i].fd == get_keychord_fd())
                    handle_keychord();
                else if (ufds[i].fd == get_signal_fd())
                    handle_signal();
            }
        }
    }

    return 0;
}
```