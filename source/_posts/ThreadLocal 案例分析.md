---
title: ThreadLocal 案例分析
toc: true
categories:
    - Java
    - 源码分析
tags:
    - ThreadLocal
---

目录
```
1. ThreadLocal简介
    1.1 ThreadLocal基础
        1.1.1 ThreadLocal和Thread的关系
        1.1.2 变量的生命周期
    1.2 可继承的ThreadLocal
2. ThreadLocal的应用案例
    2.1 解决并发问题
        2.1.1 java.lang.ThreadLocalRandom
        2.1.2 HDFS中的Statistics：实现高并发下的统计功能
    2.2 解决数据存储问题
        2.2.1 Struts2的ActionContext设计原理
        2.2.2 Spring中thread scope Bean
3. 总结
```
## 1. ThreadLocal简介
> 这篇博客主要对ThreadLocal类的基础知识和实践应用进行分析。文章的重点在于应用案例的探究，同时也会对理论基础作简单的介绍。

### 1.1 ThreadLocal基础
###### 为什么需要ThreadLocal
要理解为什么需要ThreadLocal就不得不从**线程安全**问题说起。高并发是很多领域都会遇到的非常棘手的问题，其最核心的问题在于**如何平衡高性能和数据一致性**。当我们说某个类是线程安全的时候，也就意味着该类在多线程环境下的状态保持一致性。
> 所谓的**一致性**，就是关联数据之间的逻辑关系是否正确和完整。

通过下面示例对数据一致性问题进行说明：
```
public class ThreadLocalDemo {

	public static void main(String[] args) throws InterruptedException {
		int nThreads = 10;
		final Counter counter = new Counter();
		
		ExecutorService exec = Executors.newFixedThreadPool(nThreads);
		final CountDownLatch latch = new CountDownLatch(nThreads);
		
		for(int i = 0; i < nThreads; i++){
			exec.submit(new Runnable(){
				public void run(){
					for(int i = 0; i < 10000; i++){
						counter.increase();
					}
					latch.countDown();
				}
			});
		}
		latch.await();
		System.out.println("Expected:" + nThreads * 10000 + ",Actual:" + counter.count);
	}
	static class Counter{
		int count = 0;
		
		public void  increase(){
			this.count++;
		}
	}
```
输出：
> Expected:100000,Actual:71851

可见最终变量count的状态并不符合预期的逻辑。对于并发问题来说，最简单的解决办法就是**加锁**，本质是**并发访问**到**串行访问**的改变。如下：
```
	static class Counter{
		int count = 0;
		
		public synchronized void  increase(){
			this.count++;
		}		
	}
```
输出：
> Expected:100000,Actual:100000

第一次实验中，count变量的值之所以出现不正确的情况，是因为其被多个线程同时访问，而且对某个线程来说，其它线程对变量count的操作结果，该线程是不一定可见的，这是造成count变量最终数据不一致的原因。而用**synchronized**修饰过后，**串行访问**时就不存在不可见的情况。从而保证了count变量的正确性。那么是否可以换个思路：让变量只能被一个线程访问，这不就不存在之前谈到的线程安全问题了吗？

> 让每个线程都保存一份变量的副本，该副本只会被隶属的线程操作，这也就不存在线程安全问题了。这就是ThreadLocal的由来。

#### 1.1.1 ThreadLocal和Thread的联系
在上面提到了**数据副本**,那么线程如何保存该副本的呢？其实，Thread类中有一个ThreadLocalMap类型的变量threadLocals，定义如下：
```
public class Thread implements Runnable {
    
    //。。。
    
    ThreadLocal.ThreadLocalMap threadLocals = null;
    
    //。。。    
}
```
ThreadLocalMap是ThreadLocal的一个内部类，其作用相当于一个HashMap，用于保存隶属于该线程的变量副本。下面需要考虑一个问题：**ThreadLocalMap的key和value该如何设计呢？**

从API角度来说，**ThreadLocal的作用是提供给client访问Thread中threadLocals变量的访问接口**，每个ThreadLocal都对应着一个Thread内部的变量副本。所以ThreadLocalMap中的**key**就是ThreadLocal对象（也就是该对象的hashCode），value也就是变量副本。一个对象默认的hashcode也就是该对象的引用值，这可以保证不同对象的hashcode不同。不过ThreadLocal并没有使用这一默认值，而是内部声明了一个threadLocalHashCode整型变量用以存储该对象的hashcode值：
```
public class ThreadLocal<T> {

    private final int threadLocalHashCode = nextHashCode();

    private static AtomicInteger nextHashCode =
        new AtomicInteger();

    private static final int HASH_INCREMENT = 0x61c88647;

    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }
    //。。。
}
```

**变量副本**的存储问题已经解决，那么怎么对Thread内部的threadLocals变量进行访问呢？这就要通过ThreadLocal了。下面对ThreadLocal的方法简单介绍下：

1. get()操作
```
    public T get() {
        Thread t = Thread.currentThread();//获取当前Thread对象引用
        ThreadLocalMap map = getMap(t);//从Thread对象中获取ThreadLocalMap变量
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null)
                return (T)e.value;
        }
        return setInitialValue();//如果是第一次访问，就setInitialValue进行初始化
    }
    
    private T setInitialValue() {
        //initialValue方法是protected修饰的，默认返回null，所以需要在ThreadLocal子类中进行覆盖。
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }    
```
2. set操作
```
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```
和setInitialValue几乎一致，不同的是：set操作会传入需要设置的value。而setInitialValue需要通过initialValue()获取初始值。
3. remove操作
```
     public void remove() {
         ThreadLocalMap m = getMap(Thread.currentThread());
         if (m != null)
             m.remove(this);
     }
```
#### 1.1.2 变量的生命周期
这里所的**变量**指的是存储在Thread对象中的变量副本。下面从init-service-destroy三个阶段分析下其生命周期:
1. Init
第一次调用get方法的时候完成了初始化过程。这也就是为什么需要覆盖ThreadLocal的initialValue方法。在setInitialValue方法中的createMap方法如下：
```
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```
2. Service
只要线程活着且ThreadLocal可访问即处于Service阶段。

3. Destroy
由于threadLocals变量是Thread的成员，那么当Thread对象挂了后，那么其内部的所有成员也都被gc了。此外，通过ThreadLocal提供remove方法也可以将threadLocals里的特定副本变量移除。

> ThreadLocal变量的生命周期呢？由于ThreadLocal变量通常用private static修饰，也就是属于类成员
变量。所以其生命周期当然也就和该类一致。

### 1.2 可继承的ThreadLocal
首先看个实例：
```
	static class Context {

		private static final ThreadLocal<HashMap<String,String>> CONTEXT1 = new ThreadLocal<HashMap<String,String>>(){
			protected HashMap<String,String> initialValue(){
				return new HashMap<String,String>();
			}
		};
		private static final InheritableThreadLocal<HashMap<String,String>> CONTEXT2 = new InheritableThreadLocal<HashMap<String,String>>(){
			protected HashMap<String,String> initialValue(){
				return new HashMap<String,String>();
			}
		};		
		public static HashMap<String,String> getContext1() {
			return CONTEXT1.get();
		}
		public static HashMap<String,String> getContext2() {
			return CONTEXT2.get();
		}
	}
	
	public static void main(String[] args) throws InterruptedException {
		Context.getContext1().put("name", "wqx");
		Context.getContext2().put("name", "wqx");
		Thread thread = new Thread(new Runnable(){
			@Override
			public void run() {
				System.out.println("name:" + Context.getContext1().get("name"));
				System.out.println("name:" + Context.getContext2().get("name"));
			}
		});
		thread.start();
    }
```
输出：
> name:null

> name:wqx

字面意思上理解InheritableThreadLocal即为可继承的ThreadLocal，这里的可继承的含义指的是子线程在实例化过程中，会查看当前执行线程（可以理解为父线程）的inheritableThreadLocals是否为null，如果不为null，则将该变量赋值给子线程的inheritableThreadLocals。下面是Thread类构造函数中的相关片段：
```
    Thread parent = currentThread();//当前线程，也就是执行new Thread()的线程
    if (parent.inheritableThreadLocals != null)
        this.inheritableThreadLocals =
            ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
```

## 2. ThreadLocal的应用案例
### 2.1 解决并发问题
#### 2.1.1 java.lang.ThreadLocalRandom
在Java中随机数可以用Random类，下面是java.util.Random的生成随机数的方法：
```
    protected int next(int bits) {
        long oldseed, nextseed;
        AtomicLong seed = this.seed;
        do {
            oldseed = seed.get();
            nextseed = (oldseed * multiplier + addend) & mask;
        } while (!seed.compareAndSet(oldseed, nextseed));
        return (int)(nextseed >>> (48 - bits));
    }
```
可见，其中通过CAS方式保证其线程安全性。这在高并发的环境中由于线程间的竞争必然带来一定的性能损耗。ThreadLocal此时就派上用场了，ThreadLocalRandom是通过ThreadLocal改进的用于随机数生成的工具类，每个线程单独持有一个ThreadLocalRandom对象引用，这就完全杜绝了线程间的竞争问题。
```
public class ThreadLocalRandom extends Random {
    
    //。。。
    
    private static final ThreadLocal<ThreadLocalRandom> localRandom =
        new ThreadLocal<ThreadLocalRandom>() {
            protected ThreadLocalRandom initialValue() {
                return new ThreadLocalRandom();
            }
    };
    //。。。
}
```
> ThreadLocalRandom能用于全局范围的随机数生成吗？每个线程都持有一个ThreadLocalRandom对象，生成的随机数不会重复吗？考虑到ThreadLocal的特点，理论上也就不应该将其用于全局范围，其更适合于线程独享变量的存储。But！凡事都有例外，下面看个例外的用法。

#### 2.1.2 HDFS中的Statistics：实现高并发下的统计功能
Hadoop的分布式文件系统（HDFS）是其生态的基石，MR任务中涉及到的数据输入输出都与其密切相关。对于FileSystem来说，对大量的读写操作进行统计是非常必要的。这该如何实现呢？

> 方案一：通过加锁的方式。考虑到Hadoop处理的数据体量及对数据操作的频率，加锁带来的性能损耗不可忽视，So。。。PASS！

> 方案二：ThreadLocal可以吗？对当前FileSystem进行操作的线程很多，如果只使用ThreadLocal方案的话，只能统计一个线程的操作次数，那么在汇总操作的时候必然要进行同步synchronized处理。这可行吗？判断一个方案可不可行，必须要具体业务逻辑具体分析，在本例中，statistics是用于存储**统计数据**的对象，那么对FileSystem进行操作（比如：create、mkdir、list、delete等）的同时都会记录在statistics对象中，也就是对statistics对象进行写操作，而对于统计数据的读操作比较少。所以Hadoop考虑到**写多读少**的事实，ThreadLocal方案是可以接受的。

下面是Statistics对象的部分实现：
```
public static final class Statistics {

    /**
     * Statistics data.
     * /
    public static class StatisticsData {
      volatile long bytesRead;
      volatile long bytesWritten;
      volatile int readOps;
      volatile int largeReadOps;
      volatile int writeOps;
      //。。。
    }

    //allData保存的是所有线程中StatisticsData对象的引用
    private final Set<StatisticsDataReference> allData;
    
    //ThreadLocal变量
    private final ThreadLocal<StatisticsData> threadData;
    
    public void incrementBytesWritten(long newBytes) {
      getThreadStatistics().bytesWritten += newBytes;
    }
    
    public StatisticsData getThreadStatistics() {
      StatisticsData data = threadData.get();
      if (data == null) {   //第一次统计操作时需要进行初始化，并与allData进行关联
        data = new StatisticsData();
        threadData.set(data);
        StatisticsDataReference ref =
            new StatisticsDataReference(data, Thread.currentThread());
        synchronized(this) {
          allData.add(ref);
        }
      }
      return data;
    }    
    //。。。
}
```
下面是DistributedFileSystem中删除操作的实现，可见在每次执行删除操作的时候，都会通过statistics进行记录。
```
public class DistributedFileSystem extends FileSystem {

  @Override
  public boolean delete(Path f, final boolean recursive) throws IOException {
    statistics.incrementWriteOps(1);
    // 。。。
  }
}
```
如果需要获取统计数据时，就要将所有线程内部的统计数据进行累加，这肯定需要进行同步处理的。如下所示的是获取统计数据中所有写操作的次数：
```
    public long getBytesWritten() {
      return visitAll(new StatisticsAggregator<Long>() {
        private long bytesWritten = 0;

        @Override
        public void accept(StatisticsData data) {
          bytesWritten += data.bytesWritten;
        }

        public Long aggregate() {
          return bytesWritten;
        }
      });
    }
    //加锁处理，保证统计数据的正确性
    private synchronized <T> T visitAll(StatisticsAggregator<T> visitor) {
      visitor.accept(rootData);
      for (StatisticsDataReference ref: allData) {
        StatisticsData data = ref.getData();
        visitor.accept(data);
      }
      return visitor.aggregate();
    }
```
在写多读少的环境下，这种方案可以有效的解决传统“加锁”方案带来的多线程间的竞争。Brilliant idea！

### 2.2 解决数据存储问题
#### 2.2.1 Struts2的ActionContext设计原理
Struts2是使用较为广泛的MVC框架，其关于请求响应流程的设计思路也是很新颖的。当第一次接触Struts2的时候，曾一直困惑于一个问题：**Action中的每个方法的请求参数怎么获得的**?**处理结果又是如何返回的**?在传统的Servlet中，我们可以通过函数入参HttpServletRequest对象获取请求参数，可以通过入参HttpServletResponse对象向输出流写入响应数据。而Struts2中自定义的Action的每个方法都没有入参，且处理后的响应数据也不是当作返回值返回的。

> Struts2的**最大亮点也许就是对数据流和控制流的解耦**。数据不再需要作为方法参数传入或作为返回值返回。Struts2的返回值仅仅作为控制流的标识（比如：选择哪个视图）。Struts2中数据载体就是ActionContext。不管是请求参数亦或是处理后的响应数据都被封装在ActionContext内部。开发者一般常接触的是ActionContext的子类ServletActionContext。

首先看下Struts2中几个主要组件的示意图：

![Struts2组件示意图](http://upload-images.jianshu.io/upload_images/2599999-651c2e2c3c361d94.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

ActionContext作为数据载体，与每个组件都会有数据交互，如：ActionInvocation、Interceptor、Action、Result等。这几乎涵盖了一个请求的整个生命周期。这里说的**请求的生命周期**可以泛指**处理请求的线程的生命周期**。ThreadLocal不正适合这种情况吗？下面看下com.opensymphony.xwork2.ActionContext类的部分结构：
```
public class ActionContext implements Serializable {

    //。。。
    
    static ThreadLocal<ActionContext> actionContext = new ThreadLocal<ActionContext>();
    
    private Map<String, Object> context;

    public ActionContext(Map<String, Object> context) {
        this.context = context;
    }
    
    public static ActionContext getContext() {
        return actionContext.get();
    }
    public Map<String, Object> getContextMap() {
        return context;
    }
    //。。。
```
ActionContext是典型的ThreadLocal使用案例，通过将请求处理过程中涉及到的所有参数封装进ActionContext中，从而实现了数据流和控制流的分离，这一**解耦思路**值得好好学习。Another brilliant idea！

#### 2.2.1 Spring中thread scope Bean
在Spring中，如果按照Bean的生命周期对其进行划分，那么大致可以分为这么几类：Singleton、Prototype、Request、Session、Thread Scope等。这一节主要介绍ThreadScope的Bean如何实现。经过上面的各种案例分析，这个问题就灰常容
易解决了，只需要将Bean的生命周期与Thread同步就行。ThreadLocal正合适。下面是Spring内部已经实现的方案SimpleThreadScope：
```
public class SimpleThreadScope implements Scope {

	private final ThreadLocal<Map<String, Object>> threadScope =
			new NamedThreadLocal<Map<String, Object>>("SimpleThreadScope") {
				@Override
				protected Map<String, Object> initialValue() {
					return new HashMap<String, Object>();
				}
			};

	@Override
	public Object get(String name, ObjectFactory<?> objectFactory) {
		Map<String, Object> scope = this.threadScope.get();
		Object object = scope.get(name);
		if (object == null) {
			object = objectFactory.getObject();
			scope.put(name, object);
		}
		return object;
	}

    //。。。
}
```
## 3. 总结
上面小节中分别分析了ThreadLocal的两个主要的应用领域：1.解决并发问题。2.解决数据存储问题。其中解决并发问题的本质是一种**以空间换时间的思路**，时间效率提升了，但是也存在着内存使用时的潜在溢出风险。数据存储问题主要指的是：系统中多个组件如何实现数据的交互和共享，而作为执行者的线程作为数据载体再适合不过了。虽然各种组件可以实现数据共享，但是数据在线程间是隔离的。