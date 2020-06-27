ThreadLocalMap是ThreadLocal内部类，是ThreadLocal的核心

## ThreadLocalMap源代码分析

```java
static class ThreadLocalMap {

    /** Map中的映射实体对象
         */
    static class Entry extends WeakReference<ThreadLocal> {
        /** 线程放入ThreadLocal的变量元素 */
        Object value;
        Entry(ThreadLocal k, Object v) {
            super(k);
            value = v;
        }
    }

    /** 初始容量 */
    private static final int INITIAL_CAPACITY = 16;
    /** 数组 */
    private Entry[] table;
    /** 实际存放元素个数 */
    private int size = 0;
    /** 数组扩容的阈值，默认为0 */
    private int threshold;

    /** 设置阈值大小*/
    private void setThreshold(int len) {
        threshold = len * 2 / 3;
    }
    /** 数组中元素的下一个的下标 */
    private static int nextIndex(int i, int len) {
        return ((i + 1 < len) ? i + 1 : 0);
    }
    /**数组中元素的上一个的下标*/
    private static int prevIndex(int i, int len) {
        return ((i - 1 >= 0) ? i - 1 : len - 1);
    }

    /** 构造方法 */
    ThreadLocalMap(ThreadLocal firstKey, Object firstValue) {
        table = new Entry[INITIAL_CAPACITY];
        int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
        table[i] = new Entry(firstKey, firstValue);
        size = 1;
        setThreshold(INITIAL_CAPACITY);
    }

    /** 从其他ThreadLocalMap初构造新的ThreadLocalMap */
    private ThreadLocalMap(ThreadLocalMap parentMap) {
        Entry[] parentTable = parentMap.table;
        int len = parentTable.length;
        setThreshold(len);
        table = new Entry[len];

        for (int j = 0; j < len; j++) {
            Entry e = parentTable[j];
            if (e != null) {
                ThreadLocal key = e.get();
                if (key != null) {
                    Object value = key.childValue(e.value);
                    Entry c = new Entry(key, value);
                    int h = key.threadLocalHashCode & (len - 1);
                    while (table[h] != null)
                        h = nextIndex(h, len);
                    table[h] = c;
                    size++;
                }
            }
        }
    }
    /** 
     * 存储指定的变量
     */
    private void set(ThreadLocal key, Object value) {
        Entry[] tab = table;
        int len = tab.length;
        // 计算key在数组中理论上的下标i
        int i = key.threadLocalHashCode & (len-1);
		
        // 从下标处开始遍历查找，查找过程会有2种情况：
        // 1、映射实体的key和当前给定key相同，说明找到了，直接新值覆盖旧值
        // 2、映射实体的key为null，需要整理数组，否则会有key为null位置后面部分元素可能访问不到的问题
        // 循环退出2个可能：找到了相同ThreadLocal对象、向后遍历时下一个元素的key对应的是null 跳出
        for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
            ThreadLocal k = e.get();
            if (k == key) {
                e.value = value;
                return;
            }
            // e!=null && k==null 说明key被垃圾回收了
            if (k == null) {
                // 被回收的话就需要替换掉过期的值
                replaceStaleEntry(key, value, i);
                return;
            }
        }
        
		// 走到这里，说明没有找到，直接放入下标处
        tab[i] = new Entry(key, value);
        int sz = ++size;
        // 扩容
        if (!cleanSomeSlots(i, sz) && sz >= threshold){
            rehash();
        }
    }
    
    /** 
     * 替换过期实体值
     */
    private void replaceStaleEntry(ThreadLocal key, Object value, int staleSlot) {
        Entry[] tab = table;
        int len = tab.length;
        Entry e;
        int slotToExpunge = staleSlot;
        //这个循环是当前过期值下标staleSlot处，向前遍历，这样是为了把前面所有的已被垃圾回收的也一起释放空间出来
        //这里只是key被回收，value还没被回收，entry更加没回收
        for (int i = prevIndex(staleSlot, len); (e = tab[i]) != null; i = prevIndex(i, len)){
            if (e.get() == null){
                slotToExpunge = i;
            }
        }
        
        // 从当前过期下标staleSlot开始向后遍历
        for (int i = nextIndex(staleSlot, len); (e = tab[i]) != null; i = nextIndex(i, len)) {
            ThreadLocal k = e.get();
            // 如果找到了给定key的映射实体（新值替换旧值、交换下标i和过期位置staleSlot处两个元素）
            if (k == key) {
                e.value = value;
                tab[i] = tab[staleSlot];
                tab[staleSlot] = e;
                //这里的意思就是前面的第一个for 循环(i--)往前查找的时候没有找到过期的，只有staleSlot
                //由于前面过期的对象已经通过交换位置的方式放到index=i上了，所以需要清理的位置是i，而不是传过来的staleSlot
                if (slotToExpunge == staleSlot){
                    slotToExpunge = i;
                }
                //清理过期数据
                cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
                return;
            }
            // 如果我们在第一个for 循环(i--) 向前遍历的时候没有找到任何过期的对象
            // 那么我们需要把slotToExpunge 设置为向后遍历(i++) 的第一个过期对象的位置
            // 因为如果整个数组都没有找到要设置的key 的时候，该key 会设置在该staleSlot的位置上
            // 如果数组中存在要设置的key,那么上面也会通过交换位置的时候把有效值移到staleSlot位置上
            // 综上所述，staleSlot位置上不管怎么样，存放的都是有效的值，所以不需要清理的
            if (k == null && slotToExpunge == staleSlot)
                slotToExpunge = i;
        }
		
        //如果key 在数组中没有存在，那么直接新建一个新的放进去就可以
        tab[staleSlot].value = null;
        tab[staleSlot] = new Entry(key, value);
        // 如果有其他已经过期的对象，那么需要清理它
        if (slotToExpunge != staleSlot){
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
        }
    }

	/**
	 * 清理给定位置处的过期元素
	 */
    private int expungeStaleEntry(int staleSlot) {
        Entry[] tab = table;
        int len = tab.length;
        tab[staleSlot].value = null;
        tab[staleSlot] = null;
        size--;

        Entry e;
        int i;
        for (i = nextIndex(staleSlot, len);(e = tab[i]) != null; i = nextIndex(i, len)) {
            ThreadLocal k = e.get();
            if (k == null) {
                e.value = null; //help GC
                tab[i] = null;
                size--;
            } else {
                //由于采用了开放地址法，清理的元素可能是多个冲突元素中的一个，所以需要让后面的元素往前面移动
                //迁移的原因是：开放地址法查找元素时，遇到null会停止查找了，前面k==null，不移动的话，那么后面的元素就永远访问不了
                int h = k.threadLocalHashCode & (len - 1);
                if (h != i) {
                    tab[i] = null;
                    while (tab[h] != null){
                        h = nextIndex(h, len);
                    }
                    tab[h] = e;
                }
            }
        }
        return i;
    }
    
    
    
    /** 
     * 获取元素所在的实体Entry 
     * ThreadLocalMap解决哈希冲突的方案是：开放地址法
     */
    private Entry getEntry(ThreadLocal key) {
        // 计算给定key在“理论上”映射实体Entry的位置下标i
        int i = key.threadLocalHashCode & (table.length - 1);
        Entry e = table[i];
        // 如果该位置元素不为空，且key相同，说明就是当前要找的元素
        if (e != null && e.get() == key)
            return e;
        else {
            //走到此处有2种情况：
            // 1、如果下标i处的位置为空，getEntryAfterMiss方法中会返回null，说明指定key的元素没存过
            // 2、如果下标i处的元素不为空，同时元素key与当前给定key不相同，那么按照开放地址法的规则在当前位置向后遍历查找匹配了
            return getEntryAfterMiss(key, i, e);
        }
    }

    /** 
     * 按照开放地址法规则，从数组下标i处向后查找key对应的映射实体
     */
    private Entry getEntryAfterMiss(ThreadLocal key, int i, Entry e) {
        Entry[] tab = table;
        int len = tab.length;
		
        while (e != null) {
            ThreadLocal k = e.get();
            //实体key找到了，直接返回
            if (k == key){
                return e;
            }
			//实体key为null，整理数组元素（开放地址法中必须要整理，否则会造成后续元素可能就永远访问不了的问题）
            if (k == null) {
                expungeStaleEntry(i);
            } else {
                i = nextIndex(i, len);
            }
            e = tab[i];
        }
        return null;
    }


    /** 
     * 移除指定key的映射实体
     */
    private void remove(ThreadLocal key) {
        Entry[] tab = table;
        int len = tab.length;
        int i = key.threadLocalHashCode & (len-1);
        for (Entry e = tab[i];
             e != null;
             e = tab[i = nextIndex(i, len)]) {
            if (e.get() == key) {
                e.clear();
                expungeStaleEntry(i);
                return;
            }
        }
    }
    /**
     * 清理指定位置处元素
     */
    private boolean cleanSomeSlots(int i, int n) {
        boolean removed = false;
        Entry[] tab = table;
        int len = tab.length;
        do {
            i = nextIndex(i, len);
            Entry e = tab[i];
            if (e != null && e.get() == null) {
                n = len;
                removed = true;
                i = expungeStaleEntry(i);
            }
        } while ( (n >>>= 1) != 0);
        return removed;
    }
    

    /** 扩容 */
    private void rehash() {
        expungeStaleEntries();
        if (size >= threshold - threshold / 4){
            resize();
        }  
    }

    /** 
     * ThreadLocalMap中的数组扩容
     */
    private void resize() {
        Entry[] oldTab = table;
        int oldLen = oldTab.length;
        int newLen = oldLen * 2;
        Entry[] newTab = new Entry[newLen];
        int count = 0;

        for (int j = 0; j < oldLen; ++j) {
            Entry e = oldTab[j];
            if (e != null) {
                ThreadLocal k = e.get();
                if (k == null) {
                    e.value = null; // Help the GC
                } else {
                    int h = k.threadLocalHashCode & (newLen - 1);
                    while (newTab[h] != null)
                        h = nextIndex(h, newLen);
                    newTab[h] = e;
                    count++;
                }
            }
        }
        setThreshold(newLen);
        size = count;
        table = newTab;
    }

    /** 清理数组 */
    private void expungeStaleEntries() {
        Entry[] tab = table;
        int len = tab.length;
        for (int j = 0; j < len; j++) {
            Entry e = tab[j];
            if (e != null && e.get() == null)
                expungeStaleEntry(j);
        }
    }
    
}
```

