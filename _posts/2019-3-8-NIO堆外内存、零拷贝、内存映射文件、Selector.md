1.ByteBuffer的allocateDirect方法：
    - 创建一个DirectByteBuffer对象（直接缓冲），这个对象仍位于JVM所管理的堆（Heap）中，DirectByteBuffer的构造方法如下：
DirectByteBuffer(int cap) { // package-private
​
        super(-1, 0, cap, cap);
        boolean pa = VM.isDirectMemoryPageAligned();
        int ps = Bits.pageSize();
        long size = Math.max(1L, (long)cap + (pa ? ps : 0));
        Bits.reserveMemory(size, cap);
​
        long base = 0;
        try {
            base = unsafe.allocateMemory(size);
        } catch (OutOfMemoryError x) {
            Bits.unreserveMemory(size, cap);
            throw x;
        }
        unsafe.setMemory(base, size, (byte) 0);
        if (pa && (base % ps != 0)) {
            // Round up to page boundary
            address = base + ps - (base & (ps - 1));
        } else {
            address = base;
        }
        cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
        att = null;
    }
DirectByteBuffer创建过程解析如下：
    - 首先第3行，创建了一个ByteBuffer对象，此对象位于JVM所管理的堆中，其存在的意义便是作为操作堆外内存的接口。
    - 第4至6行，会确定分配内存的大小，如果系统内存是按页对齐的，那么分配的内存大小要在capacity的基础上再加上一个内存page的大小。
    - 第7行，Bits.reserveMemory(size, cap)方法，其会在Bits类中保存分配的总内存大小和当前内存大小等信息，当再有需要分配堆外内存操作的时候，程序会更新Bits类中保存的这些信息，其实现代码如下，重要代码的具体功能在注释中给出：
static void reserveMemory(long size, int cap) {
    
    // 确定最大堆外内存大小，确定之后，memoryLimitSet属性置为true，即之后maxMemory值的大小将不再变化
    if (!memoryLimitSet && VM.isBooted()) {
        maxMemory = VM.maxDirectMemory();
        memoryLimitSet = true;
    }
​
    // 如JDK本身的注释所说，乐观做法，真正分配内存之前，假定内存是够分的，所以先调用tryReserveMemory方法
    // 先将Bits类里存储的相关数值更新，若更新成功了，则直接返回
    // optimist!
    if (tryReserveMemory(size, cap)) {
        return;
    }
    
    // 不幸的是，内存不是很充足，那么jlra.tryHandlePendingReference()会触发一次非堵塞的                       // Reference#tryHandlePending(false)，该方法会将已经被JVM垃圾回收的DirectBuffer对象的堆外内存释放
    final JavaLangRefAccess jlra = SharedSecrets.getJavaLangRefAccess();
​
    // retry while helping enqueue pending Reference objects
    // which includes executing pending Cleaner(s) which includes
    // Cleaner(s) that free direct buffer memory
    while (jlra.tryHandlePendingReference()) {
        if (tryReserveMemory(size, cap)) {
            return;
        }
    }
    
    // 如果通过虚引用的方式仍旧无法腾出足够的内存的话，就进行一次Full GC
    // 注意：如果JVM启动参数中添加了 -XX:+DisableExplicitGC的话，那么系统就无法进行手动垃圾回收，
    // 那么此处代码会失效
    // trigger VM's Reference processing
    System.gc();
​
    // Full GC之后，会再进行最多9次的分配内存（实际是改变Bits中保存的各项内存指标相关的值）
    // a retry loop with exponential back-off delays
    // (this gives VM some time to do it's job)
    boolean interrupted = false;
    try {
        long sleepTime = 1;
        int sleeps = 0;
        while (true) {
            if (tryReserveMemory(size, cap)) {
                return;
            }
            // 最多会尝试9次
            if (sleeps >= MAX_SLEEPS) {
                break;
            }
            if (!jlra.tryHandlePendingReference()) {
                try {
                    Thread.sleep(sleepTime);
                    sleepTime <<= 1;
                    sleeps++;
                } catch (InterruptedException e) {
                    interrupted = true;
                }
            }
        }
        
        // 如果还不灵的话，那么只能抱怨运气太差了，下次再来请求堆外内存吧，本次只能以一个内存溢出的错误结束
        // no luck
        throw new OutOfMemoryError("Direct buffer memory");
​
    } finally {
        if (interrupted) {
            // don't swallow interrupts
            Thread.currentThread().interrupt();
        }
    }
}
    - 第11行代码为调用native方法为buffer分配内存，其底层的实现为C的malloc方法，此内存并不是JVM管理的内存，而是由操作系统直接管理的“堆外内存”，而DirectByteBuffer利用其顶层父类Buffer中的address属性来存储该内存的基地址，借助address，DirectByteBuffer得以操作该内存区域，注意：当分配过程中出错的话，会将在此之前改变的Bits中保存的内存相关值再置回去，即Bits.unreserveMemory(size, cap)。
    - 分配内存后，会得到一个long型的变量值base，这个base便是分配的堆外内存的基地址，后面的代码将base赋给了address，如前面所述，address便是DirectByteBuffer操作堆外内存的入口。
    - 最后一个重点便是在DirectByteBuffer在创建的同时会创建一个Cleaner类的对象：cleaner = Cleaner.create(this, new Deallocator(base, size, cap))。注意：此处的创建并非常规的创建，因为Cleaner类是一种类似双向链表的结构，所以每次执行Cleaner.create的时候，其实都是向现有链表中追加了一个与当前DirectByteBuffer相对应的Cleaner对象。Cleaner类本身是PhantomReference类的一个子类，DirectByteBuffer在创建时，会将自身作为cleaner中的referent属性设置进去，同时还有一个参数referenceQueue会被传入，并且cleaner在创建时，会新启动一个线程ReferenceHandler去循环监视pending字段，此处可理解为：当DirectByteBuffer对象被回收后，cleaner链表中与之对应的cleaner对象会被赋值到pending属性上（JVM行为），稍后此对象会被回收，并把下一个要入队的元素赋值给pending（此处的下一个元素discovered是由JVM维护的，其顺序与当前链表的顺序无关，从而在ReferenceHandler的tryHandlePending方法中，触发了cleaner的clean方法，clean方法中调用了Deallocator（DirectByteBuffer的内部类）的run方法，代码如下：
public void run() {
    if (address == 0) {
        // Paranoia
        return;
    }
    unsafe.freeMemory(address);
    address = 0;
    Bits.unreserveMemory(size, capacity);
}
由代码可知，unsafe.freeMemory释放了堆外内存，至此，DirectByteBuffer的堆外内存分配及回收便完成了闭环。

2.零拷贝概念解读：

    - 以写文件为例，当用传统IO方式时，首先需将数据写入位于堆内的字节数组，然后调用JNI将数组拷贝至堆外，在堆外的内存空间中，操作系统将该数组内的数据写入磁盘，这个过程中，有一次拷贝的动作；当用NIO方式时，利用DirectByteBuffer，直接在堆外开辟内存空间，buffer利用address访问该内存空间，向其写入数据，操作系统再将数组中的数据写入磁盘，这个过程中，并没有从堆内向堆外内存区域拷贝数据的过程，故称之为零拷贝。

3.MappedByteBuffer类解析：

    - MappedByteBuffer是一种直接缓冲的buffer，它的内容映射了文件的某一区域或某个文件
    - MappedByteBuffer由FileChannel的map方法创建
    - MappedByteBuffer保存的文件映射区域内容，直到其被垃圾回收前，都是有效的
    - 在任何时候，MappedByteBuffer的部分或全部内容都可能变成不可访问的。例如：当映射的文件内容被截断了（清空），那么试图访问MappedByteBuffer内已经变得不可访问的该区域（例如在某一position执行put操作）的内容将无法改变MappedByteBuffer的内容，并且会在当时或稍后抛出异常

示例代码：
public class NioTest9 {
    public static void main(String[] args) throws Exception{
        RandomAccessFile randomAccessFile = new RandomAccessFile("src/randomAccess.txt", "rw");
        FileChannel fileChannel = randomAccessFile.getChannel();
​
        MappedByteBuffer buffer = fileChannel.map(FileChannel.MapMode.READ_WRITE, 0, 5);
​
        System.out.println("before sleep");
        Thread.sleep(10000);
        System.out.println("after sleep");
​
        buffer.put(0, (byte) 'a');
        buffer.put(3, (byte) 'e');
​
        randomAccessFile.close();
    }
}
如代码所示，在第6行MappedByteBuffer映射了src下的randomAccess.txt文件后，线程被睡眠十秒钟，在此期间，将randomAccess.txt文件的所有内容删除，睡眠结束后，向buffer写入了两个字母，但是在程序结束后，我们观察randomAccess.txt文件内容，仍然是空的，所以在文件被改变后，改变区域的内容无法被改变，因为MappedByteBuffer对应的内容已经变为不可访问的（inaccessible）

4.Selector类解析：

    - Selector是SelectableChannel对象的一种多路复用器（意为可在多个SelectableChannel之间切换）





    
