# Systrace

- Source Code
    - frameworks/native/cmds/atrace/atrace.cpp
    - system/core/include/utils/Trace.h
    - system/core/include/cutils/trace.h
    - sdk/platform-tools/systrace/systrace.py

- How to enable from Android sdk tool
    - python systrace.py --time=10 -o mynewtrace.html sched gfx view wm


- Default enable options (system property)
    - debug.atrace.tags.enableflags (0xe)
    - ATRACE_TAG_GRAPHICS (1<<1), ATRACE_TAG_INPUT (1<<2), ATRACE_TAG_VIEW (1<<3)


- How to add Systrace(Atrace) log in source code
    - java
        - Use traceBegin and traceEnd function defined in Trace.java
        - example
            - **Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "updateOomAdjLocked");**
            - **Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);**
    - C code
        - Need define ATRACE_TAG first and use ATRACE_CALL() macro
        - Example (SurfaceFlinger.cpp)
            - \#define ATRACE_TAG ATRACE_TAG_GRAPHICS
            - ATRACE_CALL();


```java
// using Trace.traceBegin and Trace.traceEnd
public void syncAll(int srcLocation) {
    Trace.traceBegin(RenderScript.TRACE_TAG, "syncAll");
    switch (srcLocation) {
    case USAGE_GRAPHICS_TEXTURE:
    case USAGE_SCRIPT:
        if ((mUsage & USAGE_SHARED) != 0) {
            copyFrom(mBitmap);
        }
        break;
    case USAGE_GRAPHICS_CONSTANTS:
    case USAGE_GRAPHICS_VERTEX:
        break;
    case USAGE_SHARED:
        if ((mUsage & USAGE_SHARED) != 0) {
            copyTo(mBitmap);
        }
        break;
    default:
        throw new RSIllegalArgumentException("Source must be exactly one usage type.");
    }
    mRS.validate();
    mRS.nAllocationSyncAll(getIDSafe(), srcLocation);
    Trace.traceEnd(RenderScript.TRACE_TAG);
}
```


```cpp
// SurfaceFlinger.cpp
#define ATRACE_TAG ATRACE_TAG_GRAPHICS

void SurfaceFlinger::eventControl(int disp, int event, int enabled) {
    ATRACE_CALL();
    getHwComposer().eventControl(disp, event, enabled);
}
```

- Framework Trace Enable
    - Change the system property

```java
// Trace.java
SystemProperties.addChangeCallback(new Runnable() {
    @Override public void run() {
        cacheEnabledTags();
    }
});

private static long cacheEnabledTags() {
    long tags = nativeGetEnabledTags();
    sEnabledTags = tags;
    return tags;
}
```

```cpp
// android_os_Trace.cpp
static JNINativeMethod gTraceMethods[] = {
    /* name, signature, funcPtr */
    { "nativeGetEnabledTags",
            "()J",
            (void*)android_os_Trace_nativeGetEnabledTags },
};

static jlong android_os_Trace_nativeGetEnabledTags(JNIEnv* env, jclass clazz) {
    return atrace_get_enabled_tags();
}
```

```cpp
// trace.h
static inline uint64_t atrace_get_enabled_tags()
{
    atrace_init();
    return atrace_enabled_tags;
}

static inline void atrace_init()
{
    if (CC_UNLIKELY(!android_atomic_acquire_load(&atrace_is_ready))) {
        atrace_setup();
    }
}
```

```cpp
void atrace_setup()
{
    pthread_once(&atrace_once_control, atrace_init_once);
}

static void atrace_init_once()
{
    atrace_marker_fd = open("/sys/kernel/debug/tracing/trace_marker", O_WRONLY);
    if (atrace_marker_fd == -1) {
        ALOGE("Error opening trace file: %s (%d)", strerror(errno), errno);
        atrace_enabled_tags = 0;
        goto done;
    }

    atrace_enabled_tags = atrace_get_property();

done:
    android_atomic_release_store(1, &atrace_is_ready);
}

static uint64_t atrace_get_property()
{
    char value[PROPERTY_VALUE_MAX];
    char *endptr;
    uint64_t tags;

    property_get("debug.atrace.tags.enableflags", value, "0");
    errno = 0;
    tags = strtoull(value, &endptr, 0);

    // Only set the "app" tag if this process was selected for app-level debug
    // tracing.
    if (atrace_is_app_tracing_enabled()) {
        tags |= ATRACE_TAG_APP;
    } else {
        tags &= ~ATRACE_TAG_APP;
    }

    return (tags | ATRACE_TAG_ALWAYS) & ATRACE_TAG_VALID_MASK;
}
```

```cpp
// Trace.h
#define ATRACE_NAME(name) android::ScopedTrace ___tracer(ATRACE_TAG, name)
// ATRACE_CALL is an ATRACE_NAME that uses the current function name.
#define ATRACE_CALL() ATRACE_NAME(__FUNCTION__)

```

- atrace.cpp

```cpp
struct TracingCategory {
    // The name identifying the category.
    const char* name;

    // A longer description of the category.
    const char* longname;

    // The userland tracing tags that the category enables.
    uint64_t tags;

    // The fname==NULL terminated list of /sys/ files that the category
    // enables.
    struct {
        // Whether the file must be writable in order to enable the tracing
        // category.
        requiredness required;

        // The path to the enable file.
        const char* path;
    } sysfiles[MAX_SYS_FILES];
};
```

```cpp
static const TracingCategory k_categories[] = {
    { "memreclaim", "Kernel Memory Reclaim", 0, {
        { REQ,      "/sys/kernel/debug/tracing/events/vmscan/mm_vmscan_direct_reclaim_begin/enable" },
        { REQ,      "/sys/kernel/debug/tracing/events/vmscan/mm_vmscan_direct_reclaim_end/enable" },
        { REQ,      "/sys/kernel/debug/tracing/events/vmscan/mm_vmscan_kswapd_wake/enable" },
        { REQ,      "/sys/kernel/debug/tracing/events/vmscan/mm_vmscan_kswapd_sleep/enable" },
    } },
};
```


```java
static bool setUpTrace()
{
    bool ok = true;

    // Set up the tracing options.
    ok &= setTraceOverwriteEnable(g_traceOverwrite);
    ok &= setTraceBufferSizeKB(g_traceBufferSizeKB);
    ok &= setGlobalClockEnable(true);

    /*
     * M: Disable by MTK
     */
    //ok &= setPrintTgidEnableIfPresent(true);
    ok &= setKernelTraceFuncs(g_kernelTraceFuncs);

    /*
     * M: Enable process name logging
     */
    ok &= setTraceRecordcmdEnable(true);

    // Set up the tags property.
    uint64_t tags = 0;
    for (int i = 0; i < NELEM(k_categories); i++) {
        if (g_categoryEnables[i]) {
            const TracingCategory &c = k_categories[i];
            tags |= c.tags;
        }
}

    ok &= setTagsProperty(tags);
    ok &= setAppCmdlineProperty(g_debugAppCmdLine);
    ok &= pokeBinderServices();

    // Disable all the sysfs enables.  This is done as a separate loop from
    // the enables to allow the same enable to exist in multiple categories.
    ok &= disableKernelTraceEvents();

    // Enable all the sysfs enables that are in an enabled category.
    for (int i = 0; i < NELEM(k_categories); i++) {
        if (g_categoryEnables[i]) {
            const TracingCategory &c = k_categories[i];
            for (int j = 0; j < MAX_SYS_FILES; j++) {
                const char* path = c.sysfiles[j].path;
                bool required = c.sysfiles[j].required == REQ;
                if (path != NULL) {
                    if (fileIsWritable(path)) {
setCategoryEnable(argv[isetCategoryEnable(argv[ible(path, true);
                    } else if (required) {
                        fprintf(stderr, "error writing file %s\n", path);
                        ok = false;
                    }
                }
            }
        }
    }

    return ok;
}
```
- atrace.cpp
- python systrace.py --time=100 -o mynewtrace.html sched gfx view wm memreclaim


```java
int main(int argc, char **argv)
{

    for (;;) {
        int ret;
        int option_index = 0;
        static struct option long_options[] = {
            {"async_start",     no_argument, 0,  0 },
            {"async_stop",      no_argument, 0,  0 },
            {"async_dump",      no_argument, 0,  0 },
            {"list_categories", no_argument, 0,  0 },
            {"poke_services",   no_argument, 0,  0 },
            {           0,                0, 0,  0 }
        };

        ret = getopt_long(argc, argv, "a:b:ck:ns:t:z",
                          long_options, &option_index);

        if (ret < 0) {
            for (int i = optind; i < argc; i++) {
                if (!setCategoryEnable(argv[i], true)) {
                    fprintf(stderr, "error enabling tracing category \"%s\"\n", argv[i]);
                    exit(1);
                }
            }
            break;
        }

    }


    bool ok = true;
    ok &= setUpTrace();
    ok &= startTrace();

    if (ok && traceStart) {
        printf("capturing trace...");
        fflush(stdout);

        // We clear the trace after starting it because tracing gets enabled for
        // each CPU individually in the kernel. Having the beginning of the trace
        // contain entries from only one CPU can cause "begin" entries without a
        // matching "end" entry to show up if a task gets migrated from one CPU to
        // another.
        ok = clearTrace();

        if (ok && !async) {
            // Sleep to allow the trace to be captured.
            struct timespec timeLeft;
            timeLeft.tv_sec = g_traceDurationSeconds;
            timeLeft.tv_nsec = 0;
            do {
                if (g_traceAborted) {
                    break;
                }
            } while (nanosleep(&timeLeft, &timeLeft) == -1 && errno == EINTR);
        }
    }

    // Stop the trace and restore the default settings.
    if (traceStop)
        stopTrace();

    if (ok && traceDump) {
        if (!g_traceAborted) {
            printf(" done\nTRACE:\n");
            fflush(stdout);
            dumpTrace();
        } else {
            printf("\ntrace aborted.\n");
            fflush(stdout);
        }
        clearTrace();
    } else if (!ok) {
        fprintf(stderr, "unable to start tracing\n");
    }

    // Reset the trace buffer size to 1.
    if (traceStop)
        cleanUpTrace();

    return g_traceAborted ? 1 : 0;
}

```
