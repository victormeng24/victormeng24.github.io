---
title: Java的四种引用类型
date: 2020-01-13 16:24:51
tags:
- Java
---
# Introduction
JVM内存是不需要程序员进行管理的，JVM本身会在适当的时候进行GC。为了辅助程序员控制Object的生命周期，Java提供了四种引用类型，按引用强度由高到低为：StrongReference(强引用）、SoftReference(软引用）、WeakReference(弱引用）、PhantomReference(虚引用)
# Reference
Reference也是一个一个对象，下面是它的部分源码，我对一些域和方法添加了中文注释：
```js
public abstract class Reference<T> {
    //持有的引用对象
    private T referent;         /* Treated specially by GC */
    //引用队列，在下面的构造器中传入
    volatile ReferenceQueue<? super T> queue;

    /* When active:   NULL
     *     pending:   this
     *    Enqueued:   next reference in queue (or this if last)
     *    Inactive:   this
     */
    //指针指向队列下一个引用
    @SuppressWarnings("rawtypes")
    volatile Reference next;

    /* When active:   next element in a discovered reference list maintained by GC (or this if last)
     *     pending:   next element in the pending list (or null if last)
     *   otherwise:   NULL
     */
    //jvm维护的discovered reference list，指向这个list的下一个元素
    transient private Reference<T> discovered;  /* used by VM */


    /* Object used to synchronize with the garbage collector.  The collector
     * must acquire this lock at the beginning of each collection cycle.  It is
     * therefore critical that any code holding this lock complete as quickly
     * as possible, allocate no new objects, and avoid calling user code.
     */
    static private class Lock { }
    private static Lock lock = new Lock();


    /* List of References waiting to be enqueued.  The collector adds
     * References to this list, while the Reference-handler thread removes
     * them.  This list is protected by the above lock object. The
     * list uses the discovered field to link its elements.
     */
    //等待加入queue的reference对象，ReferenceHandler会不断将pending对象加入到队列中。
    private static Reference<Object> pending = null;

    /* High-priority thread to enqueue pending References
     */
    private static class ReferenceHandler extends Thread {

        private static void ensureClassInitialized(Class<?> clazz) {
            try {
                Class.forName(clazz.getName(), true, clazz.getClassLoader());
            } catch (ClassNotFoundException e) {
                throw (Error) new NoClassDefFoundError(e.getMessage()).initCause(e);
            }
        }

        static {
            // pre-load and initialize InterruptedException and Cleaner classes
            // so that we don't get into trouble later in the run loop if there's
            // memory shortage while loading/initializing them lazily.
            ensureClassInitialized(InterruptedException.class);
            ensureClassInitialized(Cleaner.class);
        }

        ReferenceHandler(ThreadGroup g, String name) {
            super(g, name);
        }

        public void run() {
            while (true) {
                tryHandlePending(true);
            }
        }
    }

    /**
     * Try handle pending {@link Reference} if there is one.<p>
     * Return {@code true} as a hint that there might be another
     * {@link Reference} pending or {@code false} when there are no more pending
     * {@link Reference}s at the moment and the program can do some other
     * useful work instead of looping.
     *
     * @param waitForNotify if {@code true} and there was no pending
     *                      {@link Reference}, wait until notified from VM
     *                      or interrupted; if {@code false}, return immediately
     *                      when there is no pending {@link Reference}.
     * @return {@code true} if there was a {@link Reference} pending and it
     *         was processed, or we waited for notification and either got it
     *         or thread was interrupted before being notified;
     *         {@code false} otherwise.
     */
    //将pending的reference添加到referencequeue中，如果pending的对象是Cleaner，直接调用clean()进行回收
    static boolean tryHandlePending(boolean waitForNotify) {
        Reference<Object> r;
        Cleaner c;
        try {
            synchronized (lock) {
                if (pending != null) {
                    r = pending;
                    // 'instanceof' might throw OutOfMemoryError sometimes
                    // so do this before un-linking 'r' from the 'pending' chain...
                    c = r instanceof Cleaner ? (Cleaner) r : null;
                    // unlink 'r' from 'pending' chain
                    pending = r.discovered;
                    r.discovered = null;
                } else {
                    // The waiting on the lock may cause an OutOfMemoryError
                    // because it may try to allocate exception objects.
                    if (waitForNotify) {
                        lock.wait();
                    }
                    // retry if waited
                    return waitForNotify;
                }
            }
        } catch (OutOfMemoryError x) {
            // Give other threads CPU time so they hopefully drop some live references
            // and GC reclaims some space.
            // Also prevent CPU intensive spinning in case 'r instanceof Cleaner' above
            // persistently throws OOME for some time...
            Thread.yield();
            // retry
            return true;
        } catch (InterruptedException x) {
            // retry
            return true;
        }

        // Fast path for cleaners
        if (c != null) {
            c.clean();
            return true;
        }

        ReferenceQueue<? super Object> q = r.queue;
        if (q != ReferenceQueue.NULL) q.enqueue(r);
        return true;
    }

    static {
        ThreadGroup tg = Thread.currentThread().getThreadGroup();
        for (ThreadGroup tgn = tg;
             tgn != null;
             tg = tgn, tgn = tg.getParent());
        Thread handler = new ReferenceHandler(tg, "Reference Handler");
        /* If there were a special system-only priority greater than
         * MAX_PRIORITY, it would be used here
         */
        handler.setPriority(Thread.MAX_PRIORITY);
        handler.setDaemon(true);
        handler.start();

        // provide access in SharedSecrets
        SharedSecrets.setJavaLangRefAccess(new JavaLangRefAccess() {
            @Override
            public boolean tryHandlePendingReference() {
                return tryHandlePending(false);
            }
        });
    }

    /* -- Referent accessor and setters -- */

    /**
     * Returns this reference object's referent.  If this reference object has
     * been cleared, either by the program or by the garbage collector, then
     * this method returns <code>null</code>.
     *
     * @return   The object to which this reference refers, or
     *           <code>null</code> if this reference object has been cleared
     */
    public T get() {
        return this.referent;
    }

    /**
     * Clears this reference object.  Invoking this method will not cause this
     * object to be enqueued.
     *
     * <p> This method is invoked only by Java code; when the garbage collector
     * clears references it does so directly, without invoking this method.
     */
    public void clear() {
        this.referent = null;
    }


    /* -- Queue operations -- */

    /**
     * Tells whether or not this reference object has been enqueued, either by
     * the program or by the garbage collector.  If this reference object was
     * not registered with a queue when it was created, then this method will
     * always return <code>false</code>.
     *
     * @return   <code>true</code> if and only if this reference object has
     *           been enqueued
     */
    public boolean isEnqueued() {
        return (this.queue == ReferenceQueue.ENQUEUED);
    }

    /**
     * Adds this reference object to the queue with which it is registered,
     * if any.
     *
     * <p> This method is invoked only by Java code; when the garbage collector
     * enqueues references it does so directly, without invoking this method.
     *
     * @return   <code>true</code> if this reference object was successfully
     *           enqueued; <code>false</code> if it was already enqueued or if
     *           it was not registered with a queue when it was created
     */
    public boolean enqueue() {
        return this.queue.enqueue(this);
    }


    /* -- Constructors -- */

    Reference(T referent) {
        this(referent, null);
    }

    Reference(T referent, ReferenceQueue<? super T> queue) {
        this.referent = referent;
        this.queue = (queue == null) ? ReferenceQueue.NULL : queue;
    }

}
```
下面看下具体的实现类有何不同。
# StrongReference
StrongReference不是Reference的实现类，强引用实际上是Java的默认引用形式，使用时不需要显示定义。任何通过强引用所使用的对象不管系统资源有多紧张，Java GC都不会主动回收具有强引用的对象。
形如
```js
Object obj = new Object()
```
这样的显示声明，都是为对象建立了一个强引用，这种引用类型在我们日常coding时经常使用。
# SoftReference
```js
public class SoftReference<T> extends Reference<T> {

    /**
     * Timestamp clock, updated by the garbage collector
     */
    static private long clock;

    /**
     * Timestamp updated by each invocation of the get method.  The VM may use
     * this field when selecting soft references to be cleared, but it is not
     * required to do so.
     */
    private long timestamp;

    /**
     * Creates a new soft reference that refers to the given object.  The new
     * reference is not registered with any queue.
     *
     * @param referent object the new soft reference will refer to
     */
    public SoftReference(T referent) {
        super(referent);
        this.timestamp = clock;
    }

    /**
     * Creates a new soft reference that refers to the given object and is
     * registered with the given queue.
     *
     * @param referent object the new soft reference will refer to
     * @param q the queue with which the reference is to be registered,
     *          or <tt>null</tt> if registration is not required
     *
     */
    public SoftReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);
        this.timestamp = clock;
    }

    /**
     * Returns this reference object's referent.  If this reference object has
     * been cleared, either by the program or by the garbage collector, then
     * this method returns <code>null</code>.
     *
     * @return   The object to which this reference refers, or
     *           <code>null</code> if this reference object has been cleared
     */
    public T get() {
        T o = super.get();
        if (o != null && this.timestamp != clock)
            this.timestamp = clock;
        return o;
    }

}
```
clock是一个静态域，如注释中所说，这个值会在gc时被更新，值为当前gc的时间，timestamp当调用get()时会被赋值成clock值。
这两个值具体用途是什么呢？
以下内容部分转自掘金：
作者：做个好人君
链接：https://juejin.im/post/5bdfaeace51d4520b66399f1
来源：掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```js
size_t
ReferenceProcessor::process_discovered_reflist(
  DiscoveredList               refs_lists[],
  ReferencePolicy*             policy,
  bool                         clear_referent,
  BoolObjectClosure*           is_alive,
  OopClosure*                  keep_alive,
  VoidClosure*                 complete_gc,
  AbstractRefProcTaskExecutor* task_executor)
{
 ...
 ...
      for (uint i = 0; i < _max_num_q; i++) {
        process_phase1(refs_lists[i], policy,
                       is_alive, keep_alive, complete_gc);
      }
 ...
}

//该阶段的主要目的就是当内存足够时，将对应的SoftReference从refs_list中移除。
void
ReferenceProcessor::process_phase1(DiscoveredList&    refs_list,
                                   ReferencePolicy*   policy,
                                   BoolObjectClosure* is_alive,
                                   OopClosure*        keep_alive,
                                   VoidClosure*       complete_gc) {
  
  DiscoveredListIterator iter(refs_list, keep_alive, is_alive);
  // Decide which softly reachable refs should be kept alive.
  while (iter.has_next()) {
    iter.load_ptrs(DEBUG_ONLY(!discovery_is_atomic() /* allow_null_referent */));
    //判断引用的对象是否存活
    bool referent_is_dead = (iter.referent() != NULL) && !iter.is_referent_alive();
    //如果引用的对象已经不存活了，则会去调用对应的ReferencePolicy判断该对象是不时要被回收
    if (referent_is_dead &&
        !policy->should_clear_reference(iter.obj(), _soft_ref_timestamp_clock)) {
      if (TraceReferenceGC) {
        gclog_or_tty->print_cr("Dropping reference (" INTPTR_FORMAT ": %s"  ") by policy",
                               (void *)iter.obj(), iter.obj()->klass()->internal_name());
      }
      // Remove Reference object from list
      iter.remove();
      // Make the Reference object active again
      iter.make_active();
      // keep the referent around
      iter.make_referent_alive();
      iter.move_to_next();
    } else {
      iter.next();
    }
  }
 ...
}
```

process_discovered_reflist处理refList，判断当这个对象已经死掉且不需要删除引用时，会把这个reference从list里面删除。也就是说，留在list里面的都是需要进行gc的对象。
Policy一共有4种实现：
- NeverClearPolicy
-  AlwaysClearPolicy
-  LRUCurrentHeapPolicy
-  LRUMaxHeapPolicy

前两种很直观略过不谈，看后两种：
```js
bool LRUMaxHeapPolicy::should_clear_reference(oop p,
                                             jlong timestamp_clock) {
  jlong interval = timestamp_clock - java_lang_ref_SoftReference::timestamp(p);
  assert(interval >= 0, "Sanity check");

  // The interval will be zero if the ref was accessed since the last scavenge/gc.
  if(interval <= _max_interval) {
    return false;
  }

  return true;
}
```

```js
void LRUCurrentHeapPolicy::setup() {
  _max_interval = (Universe::get_heap_free_at_last_gc() / M) * SoftRefLRUPolicyMSPerMB;
  assert(_max_interval >= 0,"Sanity check");
}

void LRUMaxHeapPolicy::setup() {
  size_t max_heap = MaxHeapSize;
  max_heap -= Universe::get_heap_used_at_last_gc();
  max_heap /= M;

  _max_interval = max_heap * SoftRefLRUPolicyMSPerMB;
  assert(_max_interval >= 0,"Sanity check");
}
```
timestamp\_clock是上文中的静态域clock，java\_lang\_ref\_SoftReference::timestamp(p)是timestamp，这个差值当gc后调用get时为0，否则为多次gc的后与调用get()之间的差值。
max\_interval 是一个是否要gc的临界值，根据LRUCurrentHeapPolicy或者LRUMaxHeapPolicy两种策略有不同的取值逻辑。
所以SoftReference的回收机制跟下面三种因素有关:
- ReferencePolicy
-  上次gc与get()方法的时间差值
-  堆可用内存

# WeakReference
```js
public class WeakReference<T> extends Reference<T> {

    /**
     * Creates a new weak reference that refers to the given object.  The new
     * reference is not registered with any queue.
     *
     * @param referent object the new weak reference will refer to
     */
    public WeakReference(T referent) {
        super(referent);
    }

    /**
     * Creates a new weak reference that refers to the given object and is
     * registered with the given queue.
     *
     * @param referent object the new weak reference will refer to
     * @param q the queue with which the reference is to be registered,
     *          or <tt>null</tt> if registration is not required
     */
    public WeakReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);
    }

}
```
WeakReference的代码没有任何可看的内容，还是看上文提到的process\_discovered\_reflist方法：
```js
size_t
ReferenceProcessor::process_discovered_reflist(
  DiscoveredList               refs_lists[],
  ReferencePolicy*             policy,
  bool                         clear_referent,
  BoolObjectClosure*           is_alive,
  OopClosure*                  keep_alive,
  VoidClosure*                 complete_gc,
  AbstractRefProcTaskExecutor* task_executor)
{
 ...

  //Phase 1:将所有不存活但是还不能被回收的软引用从refs_lists中移除（只有refs_lists为软引用的时候，这里policy才不为null）
  if (policy != NULL) {
    if (mt_processing) {
      RefProcPhase1Task phase1(*this, refs_lists, policy, true /*marks_oops_alive*/);
      task_executor->execute(phase1);
    } else {
      for (uint i = 0; i < _max_num_q; i++) {
        process_phase1(refs_lists[i], policy,
                       is_alive, keep_alive, complete_gc);
      }
    }
  } else { // policy == NULL
    assert(refs_lists != _discoveredSoftRefs,
           "Policy must be specified for soft references.");
  }

  // Phase 2:
  // 移除所有指向对象还存活的引用
  if (mt_processing) {
    RefProcPhase2Task phase2(*this, refs_lists, !discovery_is_atomic() /*marks_oops_alive*/);
    task_executor->execute(phase2);
  } else {
    for (uint i = 0; i < _max_num_q; i++) {
      process_phase2(refs_lists[i], is_alive, keep_alive, complete_gc);
    }
  }

  // Phase 3:
  // 根据clear_referent的值决定是否将不存活对象回收
  if (mt_processing) {
    RefProcPhase3Task phase3(*this, refs_lists, clear_referent, true /*marks_oops_alive*/);
    task_executor->execute(phase3);
  } else {
    for (uint i = 0; i < _max_num_q; i++) {
      process_phase3(refs_lists[i], clear_referent,
                     is_alive, keep_alive, complete_gc);
    }
  }

  return total_list_count;
}
void
ReferenceProcessor::process_phase3(DiscoveredList&    refs_list,
                                   bool               clear_referent,
                                   BoolObjectClosure* is_alive,
                                   OopClosure*        keep_alive,
                                   VoidClosure*       complete_gc) {
  ResourceMark rm;
  DiscoveredListIterator iter(refs_list, keep_alive, is_alive);
  while (iter.has_next()) {
    iter.update_discovered();
    iter.load_ptrs(DEBUG_ONLY(false /* allow_null_referent */));
    if (clear_referent) {
      // NULL out referent pointer
      //将Reference的referent字段置为null，之后会被GC回收
      iter.clear_referent();
    } else {
      // keep the referent around
      //标记引用的对象为存活，该对象在这次GC将不会被回收
      iter.make_referent_alive();
    }
	...
  }
    ...
}
```
不管是弱引用还是其他引用类型，将字段referent置null的操作都发生在process\_phase3中，而具体行为是由clear\_referent的值决定的。而clear\_referent的值则和引用类型相关。
```js
ReferenceProcessorStats ReferenceProcessor::process_discovered_references(
  BoolObjectClosure*           is_alive,
  OopClosure*                  keep_alive,
  VoidClosure*                 complete_gc,
  AbstractRefProcTaskExecutor* task_executor,
  GCTimer*                     gc_timer) {
  NOT_PRODUCT(verify_ok_to_handle_reflists());
	...
  //process_discovered_reflist方法的第3个字段就是clear_referent
  // Soft references
  size_t soft_count = 0;
  {
    GCTraceTime tt("SoftReference", trace_time, false, gc_timer);
    soft_count =
      process_discovered_reflist(_discoveredSoftRefs, _current_soft_ref_policy, true,
                                 is_alive, keep_alive, complete_gc, task_executor);
  }

  update_soft_ref_master_clock();

  // Weak references
  size_t weak_count = 0;
  {
    GCTraceTime tt("WeakReference", trace_time, false, gc_timer);
    weak_count =
      process_discovered_reflist(_discoveredWeakRefs, NULL, true,
                                 is_alive, keep_alive, complete_gc, task_executor);
  }

  // Final references
  size_t final_count = 0;
  {
    GCTraceTime tt("FinalReference", trace_time, false, gc_timer);
    final_count =
      process_discovered_reflist(_discoveredFinalRefs, NULL, false,
                                 is_alive, keep_alive, complete_gc, task_executor);
  }

  // Phantom references
  size_t phantom_count = 0;
  {
    GCTraceTime tt("PhantomReference", trace_time, false, gc_timer);
    phantom_count =
      process_discovered_reflist(_discoveredPhantomRefs, NULL, false,
                                 is_alive, keep_alive, complete_gc, task_executor);
  }
	...
}
```
可以看到，对于Soft references和Weak references clear\_referent字段传入的都是true，这也符合我们的预期：对象不可达后，引用字段就会被置为null，然后对象就会被回收（对于软引用来说，如果内存足够的话，在Phase 1，相关的引用就会从refs\_list中被移除，到Phase 3时refs\_list为空集合）。
但对于Final references和 Phantom references，clear\_referent字段传入的是false，也就意味着被这两种引用类型引用的对象，如果没有其他额外处理，只要Reference对象还存活，那引用的对象是不会被回收的。接下来看看Phantom references。

# PhantomReference
```js
public class PhantomReference<T> extends Reference<T> {

    /**
     * Returns this reference object's referent.  Because the referent of a
     * phantom reference is always inaccessible, this method always returns
     * <code>null</code>.
     *
     * @return  <code>null</code>
     */
    public T get() {
        return null;
    }

    /**
     * Creates a new phantom reference that refers to the given object and
     * is registered with the given queue.
     *
     * <p> It is possible to create a phantom reference with a <tt>null</tt>
     * queue, but such a reference is completely useless: Its <tt>get</tt>
     * method will always return null and, since it does not have a queue, it
     * will never be enqueued.
     *
     * @param referent the object the new phantom reference will refer to
     * @param q the queue with which the reference is to be registered,
     *          or <tt>null</tt> if registration is not required
     */
    public PhantomReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);
    }

}
```
虚引用的get()永远返回null，而PhantomReference指向的对象，如果不使用clear解除虚引用的话，这个对象是永远不会回收的。
# Summary
总结一下Java的引用类型：
1. 软引用的对象回收策略与引用对象get()的时间、堆内存大的大小、ReferencePolicy都有关系。
2. 虚引用并非不会影响对象的生命周期，如果不做任何处理，被虚引用指向的对象永远不会被gc。