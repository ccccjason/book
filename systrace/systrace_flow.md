# ATRACE_CALL

- Trace.h

```cpp
// Trace.h
class ScopedTrace
{
public:
    inline ScopedTrace(uint64_t tag, const char* name)
        : mTag(tag)
    {
        atrace_begin(mTag, name);
    }

    inline ~ScopedTrace()
    {
        atrace_end(mTag);
    }

private:
    uint64_t mTag;
};
```
- trace.h

```cpp
static inline void atrace_begin(uint64_t tag, const char* name)
{
    if (CC_UNLIKELY(atrace_is_tag_enabled(tag))) {
        char buf[1024];
        size_t len;

        //  pid , function name
        len = snprintf(buf, 1024, "B|%d|%s", getpid(), name);
        write(atrace_marker_fd, buf, len);
    }
}

static inline void atrace_end(uint64_t tag)
{
    if (CC_UNLIKELY(atrace_is_tag_enabled(tag))) {
        char c = 'E';
        write(atrace_marker_fd, &c, 1);
    }
}
```

```cpp
// Trace.h
#define ATRACE_NAME(name) android::ScopedTrace ___tracer(ATRACE_TAG, name)
#define ATRACE_CALL() ATRACE_NAME(__FUNCTION__)
```

```cpp
// trace.h
#define ATRACE_TAG_NEVER            0
#define ATRACE_TAG ATRACE_TAG_NEVER
```

- main.cpp

```cpp
int main(int argc, char *argv[])
{
    // ATRACE_CALL();  展開後如下
    android::ScopedTrace ___tracer(0, __FUNCTION__);
    return 0;
}

```


end
