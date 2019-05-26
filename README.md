# Android.Optimization-notes
Android的性能优化总结~

## 开发艺术探索之Android性能优化
> 首先做为嵌入式设备 不管是在内存还是在CPU上都会有一定的限制 过多的内存会OOM 长时间占用CPU会无反应ANR 所以需要一系列的优化方案    
主要包括：布局优化、绘制优化、内存泄漏优化、响应速度优化、ListView优化、Bitmap优化、线程优化等  同时还介绍了ANR日志的分析方法

### 1. Android的性能优化方法

**1.1 布局优化** 
> 先是遍历子View树，然后测量、layout、draw。每一步测量都是耗时的，view树越深，耗时越多 所以布局的方面就是减少层级 这样绘制界面工作量就少了    
* 删除--删除无用的控件和层级    
* 选择--选择较低性能的VG，比如Linear和Relative相比较则优先Linear 因为明显Relative更复杂 但是也要避免嵌套多层   
* 标签--采用标签和ViewStub    
> * include标签：布局的重用 只支持android:layout开头的属性 这样要求android:layoutwidth和android:layout_height必须存在，否则android:layout属性无法生效    
> * merge标签:自如其名一手合并布局就是说比如，MyView明明可以很好的承载两个TextView，为什么还要多一个ViewGroup承载呢？这时候一个Merge就干掉了直接给MyView承担了   又或者线性竖直布局包含线性竖直布局那里面的明显也可以被干掉    
> * ViewStub:说白了就是给一些错误的界面的 一般也不会一开始加载出来 这时候就可以来一首隐藏
` ((ViewStub)findViewById(R.id.stub_import)).setVisibility(View.VISIBLE); `

**1.2 绘制优化** 
> 也就是传说中的onDraw 这中间明显也要避免大量的操作    
* 首先内部得很少很少有局部对象 因为onDraw是会不停的调用的 要不然一大堆临时对象不炸就怪了OOM频繁gc   
* 如上所示不能耗时操作  因为绘制嘛16ms完不成就凉了 所以超过就废

**1.3 内存泄露优化** 
> 两大点：不能长生命周期的持有短周期的也就是Handler代码的书写上 另外就是MAT工具可以找到问题    
* 静态变量导致的：比如一个静态Context持有了Activity那这个context没有结束就会导致凉凉 Activity一直无法回收   
* 单例模式导致的：修改为单例中context不再持有Activity的context而是持有Application的context即可，因为Application本来就是单例，所以这样就不会存在内存泄漏的的现象了  生命周期比Activity要长  这样一来就导致Activity对象无法及时被释放   
```
private Context context;
private static LoginManager manager;

public static LoginManager getInstance(Context context) {
    if (manager == null)
        manager = new LoginManager(context);
    return manager;
}

private LoginManager(Context context) {
    this.context = context.getApplicationContext();     //context为application的
}
```

* Handler中handleMessage传入msg如果耗时操作就凉了 而静态的存在不依赖于外部类而还是弱引用就可以方便回收了
```
/**
     * 通过静态的Handler和弱引用activity来实现防内存泄露
     * mWeakReference = new WeakReference<>(activity);
     */
    public static class DiglettHandler extends Handler{
        private static final int RANDOM_NUBER = 500;
        public final WeakReference<DiglettActivity> mWeakReference;

        public DiglettHandler(DiglettActivity activity) {
            mWeakReference = new WeakReference<>(activity);
        }
```

**1.4 响应速度优化和ANR日志分析**   
> 响应速度优化的核心思想就是避免在主线程中去做耗时操作，将耗时操作放在其他线程当中去执行 也就是子线程   
Activity如果5秒无法响应屏幕触摸事件或者键盘输入事件就会触发ANR，而BroadcastReceiver如果10秒还未执行完操作也会出现ANR    

* 当一个进程发生ANR以后系统会在/data/anr的目录下创建一个文件traces.txt，通过分析该文件就能定位出ANR的原因      
* 模拟ANR：
```

@Override
protected void onCreate(Bundle savedInstanceState) {
   super.onCreate(savedInstanceState);
   setContentView(R.layout.activity_main);
   // 以下代码是为了模拟一个ANR的场景来分析日志
   new Thread(new Runnable() {
       @Override
       public void run() {
           testANR();
       }
   }).start();
   SystemClock.sleep(10);
   initView();
}
/**
*  以下两个方法用来模拟出一个稍微不好发现的ANR
*/
private synchronized void testANR(){
   SystemClock.sleep(3000 * 1000);
}
private synchronized void initView(){
```

* 这样会出现ANR, 然后导出/data/anr/straces.txt文件. 因为内容比较多只贴出关键部分     
```

DALVIK THREADS (15):
"main" prio=5 tid=1 Blocked
  | group="main" sCount=1 dsCount=0 obj=0x73db0970 self=0xf4306800
  | sysTid=19949 nice=0 cgrp=apps sched=0/0 handle=0xf778d160
  | state=S schedstat=( 151056979 25055334 199 ) utm=5 stm=9 core=1 HZ=100
  | stack=0xff5b2000-0xff5b4000 stackSize=8MB
  | held mutexes=
  at com.szysky.note.androiddevseek_15.MainActivity.initView(MainActivity.java:0)
  - waiting to lock <0x2fbcb3de> (a com.szysky.note.androiddevseek_15.MainActivity)     
  //看出initView方法正在等待一个锁<0x2fbcb3de>  而锁的类型是一个MainActivity对象
  - held by thread 15
  //这个锁已经被线程id为15(tid=15)的线程持有了. 接下来找一下线程15
  at com.szysky.note.androiddevseek_15.MainActivity.onCreate(MainActivity.java:42)
  //看出最后指明了ANR发生的位置在ManiActivity的42行
```

* 接下来找一下线程15   
```
"Thread-404" prio=5 tid=15 Sleeping
  | group="main" sCount=1 dsCount=0 obj=0x12c00f80 self=0xeb95bc00
  | sysTid=19985 nice=0 cgrp=apps sched=0/0 handle=0xef34be80
  | state=S schedstat=( 391248 0 1 ) utm=0 stm=0 core=2 HZ=100
  | stack=0xe2bfe000-0xe2c00000 stackSize=1036KB
  | held mutexes=
  at java.lang.Thread.sleep!(Native method)
  - sleeping on <0x2e3896a7> (a java.lang.Object)
  at java.lang.Thread.sleep(Thread.java:1031)
  - locked <0x2e3896a7> (a java.lang.Object)
  at java.lang.Thread.sleep(Thread.java:985)
  at android.os.SystemClock.sleep(SystemClock.java:120)
  at com.szysky.note.androiddevseek_15.MainActivity.testANR(MainActivity.java:50)
  - locked <0x2fbcb3de> (a com.szysky.note.androiddevseek_15.MainActivity)
```

> tid = 15 就是线程15 相关信息如上, 首行已经标出线程的状态为Sleeping, 原因在50行, 就是SystemClock.sleep(3000 * 1000);这句话. 也就是testANR(). 而最后一行也表明了持有的locked<0x2fbcb3de>就是主线程在等待的那个锁对象.

**1.5 ListView优化和Bitmap优化** 

* ListView/GridView优化：
> * 采用ViewHolder避免在频繁的getView   
> * 其次通过列表的滑动状态来控制任务的执行频率，比如快速滑动时不是和开启大量异步任务  说白了就是滑动监听

* Bitmap优化：主要是想是根据需要对图片进行采样显示

**1.6 线程优化** 

* 主要思想就是采用线程池, 避免程序中存在大量的Thread. 线程池可以重用内部的线程, 避免了线程创建和销毁的性能开销. 同时线程池还能有效的控制线程的最大并发数, 避免了大量线程因互相抢占系统资源从而导致阻塞现象的发生

### 2. 内存泄漏分析工具MAT
> MAT全程Eclipse Memory Analyzer, 是一个内存泄漏分析工具.  

* 打开MAT通过菜单打开转换后的这个文件. 这里常用的就有两个   

> * Histogram: 也就是矩形图 可以直观的看出内存中不同类型的buffer的数量和占用内存大小  
> * Dominator Tree: 最舒服的树  表达关系哦 把内存中的对象按照从大到小的顺序进行排序, 并且可以分析对象之间的引用关系, 内存泄漏分析就是通过这个完成的

* 分析内存泄漏的时候需要分析Dominator Tree里面的内存信息, 一般会不直接显示出来, 可以按照**从大到小的**顺序去排查一遍. 如果发生了了泄漏, 那么在泄漏对象处**右键单击Path To GC Roots->exclude wake/soft references.** 可以看到最终是什么对象导致的无法释放. 刚才的操作之所以排除软引用和弱引用是因为,大部分情况下这两种类型都可以被gc回收掉,所以基本也就不会造成内存泄漏   

* 同样这里也可以使用**搜索功能**, 假如我们手动模拟了内存泄漏, 泄漏的对象就是Activity那么我们back退出重进循环几次, 会发现其实很多个Activit对象


