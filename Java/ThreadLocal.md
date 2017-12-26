### ThreadLocal

1. ThreadLocal帮助线程存取“线程本地”变量，此变量仅在此线程可见。
2. ThreadLocal不直接持有数据，只是协助Thread存取数据，ThreadLocal操作的是当前Thread中成员变量。
3. Thread中能存多个本地对象，存储的对象与ThreadLocal一一对应，ThreadLocal作为key，每个key对应一个数据对象，即每个threadLocal只处理一个数据。

#### get
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
    
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }


首先获取当前线程t，从线程t中获取成员变量threadLocals，在从threadLocals中获取对应的值。
从代码可以看出get()方法返回的是当前线程的成员变量中的值。如果当前线程中还没有存储数据即返回值为null时则返回默认初始值。

#### set
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
    
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }


首先获取当前线程t，从线程t中获取成员变量threadLocals，将需存储的值存储在threadLocals中。

### 数据存储
Thread中存储本地数据的成员变量为ThreadLocal.ThreadLocalMap threadLocals;


### wu



