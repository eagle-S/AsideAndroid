
    // Adjustment used in certain places where we don't know it yet.
    // (Generally this is something that is going to be cached, but we
    // don't know the exact value in the cached range to assign yet.)
    static final int UNKNOWN_ADJ = 16;

    // This is a process only hosting activities that are not visible,
    // so it can be killed without any disruption.
    static final int CACHED_APP_MAX_ADJ = 15;
    static final int CACHED_APP_MIN_ADJ = 9;

    // The B list of SERVICE_ADJ -- these are the old and decrepit
    // services that aren't as shiny and interesting as the ones in the A list.
    static final int SERVICE_B_ADJ = 8;

    // This is the process of the previous application that the user was in.
    // This process is kept above other things, because it is very common to
    // switch back to the previous app.  This is important both for recent
    // task switch (toggling between the two top recent apps) as well as normal
    // UI flow such as clicking on a URI in the e-mail app to view in the browser,
    // and then pressing back to return to e-mail.
    static final int PREVIOUS_APP_ADJ = 7;

    // This is a process holding the home application -- we want to try
    // avoiding killing it, even if it would normally be in the background,
    // because the user interacts with it so much.
    static final int HOME_APP_ADJ = 6;

    // This is a process holding an application service -- killing it will not
    // have much of an impact as far as the user is concerned.
    static final int SERVICE_ADJ = 5;

    // This is a process with a heavy-weight application.  It is in the
    // background, but we want to try to avoid killing it.  Value set in
    // system/rootdir/init.rc on startup.
    static final int HEAVY_WEIGHT_APP_ADJ = 4;

    // This is a process currently hosting a backup operation.  Killing it
    // is not entirely fatal but is generally a bad idea.
    static final int BACKUP_APP_ADJ = 3;

    // This is a process only hosting components that are perceptible to the
    // user, and we really want to avoid killing them, but they are not
    // immediately visible. An example is background music playback.
    static final int PERCEPTIBLE_APP_ADJ = 2;

    // This is a process only hosting activities that are visible to the
    // user, so we'd prefer they don't disappear.
    static final int VISIBLE_APP_ADJ = 1;

    // This is the process running the current foreground app.  We'd really
    // rather not kill it!
    static final int FOREGROUND_APP_ADJ = 0;

    // This is a system persistent process, such as telephony.  Definitely
    // don't want to kill it, but doing so is not completely fatal.
    static final int PERSISTENT_PROC_ADJ = -12;

    // The system process runs at the default adjustment.
    static final int SYSTEM_ADJ = -16;

    // Special code for native processes that are not being managed by the system (so
    // don't have an oom adj assigned by the system).
    static final int NATIVE_ADJ = -17;


/sys/module/lowmemorykiller/parameters/adj
58,176,294,411,529,1000

/sys/module/lowmemorykiller/parameters/minfree
14336,16384,20480,24576,28672,32768

/proc/<pid>/

/rootdir/init.rc:
on property:sys.sysctl.extra_free_kbytes=*
    write /proc/sys/vm/extra_free_kbytes ${sys.sysctl.extra_free_kbytes}

ActivityManagerService
updateOomAdjLocked：更新adj，当目标进程为空或者被杀则返回false；否则返回true;
computeOomAdjLocked：计算adj，返回计算后RawAdj值;
applyOomAdjLocked：应用adj，当需要杀掉目标进程则返回false；否则返回true。



    private final boolean updateOomAdjLocked(ProcessRecord app, int cachedAdj,
            ProcessRecord TOP_APP, boolean doingAll, boolean reportingProcessState, long now)


    final boolean updateOomAdjLocked(ProcessRecord app) {
        return updateOomAdjLocked(app, false);
    }

    final boolean updateOomAdjLocked(ProcessRecord app, boolean doingProcessState)

    final void updateOomAdjLocked()