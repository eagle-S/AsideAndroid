开发者选项->选择运行环境->使用 ART

代码:
Y:\kitkat_T3\android\packages\apps\Settings\src\com\android\settings/DevelopmentSettings.java

    public boolean onPreferenceChange(Preference preference, Object newValue) {
        if (SELECT_RUNTIME_KEY.equals(preference.getKey())) {
            final String oldRuntimeValue = VMRuntime.getRuntime().vmLibrary();
            final String newRuntimeValue = newValue.toString();
            if (!newRuntimeValue.equals(oldRuntimeValue)) {
                final Context context = getActivity();
                final AlertDialog.Builder builder = new AlertDialog.Builder(getActivity());
                builder.setMessage(context.getResources().getString(R.string.select_runtime_warning_message,
                                                                    oldRuntimeValue, newRuntimeValue));
                builder.setPositiveButton(android.R.string.ok, new OnClickListener() {
                    @Override
                    public void onClick(DialogInterface dialog, int which) {
                        SystemProperties.set(SELECT_RUNTIME_PROPERTY, newRuntimeValue);
                        pokeSystemProperties();
                        PowerManager pm = (PowerManager)
                                context.getSystemService(Context.POWER_SERVICE);
                        pm.reboot(null);
                    }
                });
                builder.setNegativeButton(android.R.string.cancel, new OnClickListener() {
                    @Override
                    public void onClick(DialogInterface dialog, int which) {
                        updateRuntimeValue();
                    }
                });
                builder.show();
            }
            return true;
        }

        .....
    }

    <string-array name="select_runtime_summaries">
        <item msgid="6412880178297884701">"使用 Dalvik"</item>
        <item msgid="5131846588686178907">"使用 ART"</item>
        <item msgid="4530003713865319928">"使用 ART 调试版本"</item>
    </string-array>

    <string-array name="select_runtime_values" translatable="false" >
        <item>libdvm.so</item>
        <item>libart.so</item>
        <item>libartd.so</item>
    </string-array>

运行环境有以上三种，选择运行环境确定后，会重新设置"persist.sys.dalvik.vm.lib"系统属性值，对应的值是各运行环境用到的so库名称