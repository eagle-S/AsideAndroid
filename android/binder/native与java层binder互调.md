### java层调native-binder

WindowManagerService.java#performEnableScreen

    try {
        IBinder surfaceFlinger = ServiceManager.getService("SurfaceFlinger");
        if (surfaceFlinger != null) {
            //Slog.i(TAG, "******* TELLING SURFACE FLINGER WE ARE BOOTED!");
            Parcel data = Parcel.obtain();
            data.writeInterfaceToken("android.ui.ISurfaceComposer");
            surfaceFlinger.transact(IBinder.FIRST_CALL_TRANSACTION, // BOOT_FINISHED
                                    data, null, 0);
            data.recycle();
        }
        Intent it = new Intent(Intent.ACTION_UI_BOOT_FINISH);
        mContext.sendBroadcast(it);
    } catch (RemoteException ex) {
        Slog.e(TAG, "Boot completed: SurfaceFlinger is dead!");
    }


### native层调java-binder

PermissionCache.cpp#checkPermission
IServiceManager.cpp#checkPermission

bool checkPermission(const String16& permission, pid_t pid, uid_t uid)
{
    sp<IPermissionController> pc;
    gDefaultServiceManagerLock.lock();
    pc = gPermissionController;
    gDefaultServiceManagerLock.unlock();
    
    int64_t startTime = 0;

    while (true) {
        if (pc != NULL) {
            bool res = pc->checkPermission(permission, pid, uid);
            if (res) {
                if (startTime != 0) {
                    ALOGI("Check passed after %d seconds for %s from uid=%d pid=%d",
                            (int)((uptimeMillis()-startTime)/1000),
                            String8(permission).string(), uid, pid);
                }
                return res;
            }
            
            // Is this a permission failure, or did the controller go away?
            if (pc->asBinder()->isBinderAlive()) {
                ALOGW("Permission failure: %s from uid=%d pid=%d",
                        String8(permission).string(), uid, pid);
                return false;
            }
            
            // Object is dead!
            gDefaultServiceManagerLock.lock();
            if (gPermissionController == pc) {
                gPermissionController = NULL;
            }
            gDefaultServiceManagerLock.unlock();
        }
    
        // Need to retrieve the permission controller.
        sp<IBinder> binder = defaultServiceManager()->checkService(_permission);
        if (binder == NULL) {
            /* Add begin for BootAnimation only can play after check passed, 
             * but 'permission' service just work 6s later. yinqinwu ,20190617*/
            if (uid == AID_SYSTEM) {
                return true;
            }
            /* Add end for BootAnimation,yinqinwu ,20190617*/

            // Wait for the permission controller to come back...
            if (startTime == 0) {
                startTime = uptimeMillis();
                ALOGI("Waiting to check permission %s from uid=%d pid=%d",
                        String8(permission).string(), uid, pid);
            }
            sleep(1);
        } else {
            pc = interface_cast<IPermissionController>(binder);
            // Install the new permission controller, and try again.        
            gDefaultServiceManagerLock.lock();
            gPermissionController = pc;
            gDefaultServiceManagerLock.unlock();
        }
    }
}
