###  作用
监听系统进程状态变化

package android.app;

/** {@hide} */
oneway interface IProcessObserver {

    void onForegroundActivitiesChanged(int pid, int uid, boolean foregroundActivities);
    void onProcessStateChanged(int pid, int uid, int procState);
    void onProcessDied(int pid, int uid);

}

### 常用方式

    ActivityManager am = (ActivityManager)context.getSystemService(Context.ACTIVITY_SERVICE);
    try {
        ActivityManagerNative.getDefault().registerProcessObserver(mProcessObserver);
    } catch (RemoteException e) {
    }

    private final IProcessObserver mProcessObserver;

    private class ProcessObserver extends IProcessObserver.Stub {

        @Override
        public void onForegroundActivitiesChanged(int pid, int uid, boolean foregroundActivities)
                throws RemoteException {
        }

        @Override
        public void onImportanceChanged(int pid, int uid, int importance) throws RemoteException {
            
        }

        @Override
        public void onProcessDied(int pid, int uid) throws RemoteException {

        }
    }


### 回调实现
    ActivityManagerService.java

    private void dispatchProcessesChanged() {
        int N;
        synchronized (this) {
            N = mPendingProcessChanges.size();
            if (mActiveProcessChanges.length < N) {
                mActiveProcessChanges = new ProcessChangeItem[N];
            }
            mPendingProcessChanges.toArray(mActiveProcessChanges);
            mAvailProcessChanges.addAll(mPendingProcessChanges);
            mPendingProcessChanges.clear();
            if (DEBUG_PROCESS_OBSERVERS) Slog.i(TAG, "*** Delivering " + N + " process changes");
        }

        int i = mProcessObservers.beginBroadcast();
        while (i > 0) {
            i--;
            final IProcessObserver observer = mProcessObservers.getBroadcastItem(i);
            if (observer != null) {
                try {
                    for (int j=0; j<N; j++) {
                        ProcessChangeItem item = mActiveProcessChanges[j];
                        if ((item.changes&ProcessChangeItem.CHANGE_ACTIVITIES) != 0) {
                            if (DEBUG_PROCESS_OBSERVERS) Slog.i(TAG, "ACTIVITIES CHANGED pid="
                                    + item.pid + " uid=" + item.uid + ": "
                                    + item.foregroundActivities);
                            observer.onForegroundActivitiesChanged(item.pid, item.uid,
                                    item.foregroundActivities);
                        }
                        if ((item.changes&ProcessChangeItem.CHANGE_PROCESS_STATE) != 0) {
                            if (DEBUG_PROCESS_OBSERVERS) Slog.i(TAG, "PROCSTATE CHANGED pid="
                                    + item.pid + " uid=" + item.uid + ": " + item.processState);
                            observer.onProcessStateChanged(item.pid, item.uid, item.processState);
                        }
                    }
                } catch (RemoteException e) {
                }
            }
        }
        mProcessObservers.finishBroadcast();
    }

    private void dispatchProcessDied(int pid, int uid) {
        int i = mProcessObservers.beginBroadcast();
        while (i > 0) {
            i--;
            final IProcessObserver observer = mProcessObservers.getBroadcastItem(i);
            if (observer != null) {
                try {
                    observer.onProcessDied(pid, uid);
                } catch (RemoteException e) {
                }
            }
        }
        mProcessObservers.finishBroadcast();
    }


### 使用实例
Android 5.0
/frameworksbase/core/java/android/app/AppImportanceMonitor.java

Android 8.0
/packages/services/Car/service/src/com/android/car/monitoring/CarMonitoringService.java