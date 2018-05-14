开发者选项中:"显示CPU 使用情况"
Settings/src/com/android/settings/DevelopmentSettings.java

    private void updateCpuUsageOptions() {
        updateSwitchPreference(mShowCpuUsage, Settings.Global.getInt(getActivity().getContentResolver(),
                Settings.Global.SHOW_PROCESSES, 0) != 0);
    }

    private void writeCpuUsageOptions() {
        boolean value = mShowCpuUsage.isChecked();
        Settings.Global.putInt(getActivity().getContentResolver(),
                Settings.Global.SHOW_PROCESSES, value ? 1 : 0);
        Intent service = (new Intent())
                .setClassName("com.android.systemui", "com.android.systemui.LoadAverageService");
        if (value) {
            getActivity().startService(service);
        } else {
            getActivity().stopService(service);
        }
    }

"显示CPU 使用情况"打开和关闭直接是启动和关闭服务：com.android.systemui/com.android.systemui.LoadAverageService

设置中"进程统计信息"
Settings/src/com/android/settings/applications/ProcessStatsUi.java
Settings/src/com/android/settings/applications/ProcessStatsDetail.java
Settings/src/com/android/settings/applications/ProcessStatsMemDetail.java

设置-> 设备-存储





