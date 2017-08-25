---
title: 排查CPU偏高引起的一点思考
date: 2017-08-25 15:19:33
tags: [Java,性能]
categories: 思考
---

公司某个业务总是发现机器CPU偏高，要求运维扩容，结果扩容后，过了一段时间CPU还是涨了上来，又要求扩容。运维觉得老是这么扩容不是办法，就拉我一起看看问题出在哪儿。


#### 问题定位

在另外一篇文章里，介绍了如何使用[perf排查cpu偏高的问题](/2017/08/09/问题排查一次诡异的rt拉高现象/),接到问题后，马上使用perf map agent看了看，很明显就找出来耗CPU的地方：

```
Samples: 1M of event 'cpu-clock', Event count (approx.): 23964712980
  40.41%  perf-31427.map      [.] Lcom/weidian/wdp/slf4j/utils/CommonUtil;::clearMethodPerformanceMap                        ◆
  38.06%  perf-31427.map      [.] Lcom/weidian/wdp/slf4j/utils/CommonUtil;::getResourceInvokeInfo                            ▒
   1.80%  libjvm.so           [.] StringTable::intern(Handle, unsigned short*, int, Thread*)                                 ▒
   1.17%  libjvm.so           [.] java_lang_String::equals(oopDesc*, unsigned short*, int)                                   ▒
   1.01%  libjvm.so           [.] Symbol::as_klass_external_name() const                                                     ▒
   0.71%  libjvm.so           [.] Method::line_number_from_bci(int) const                                                    ▒
   0.70%  libjvm.so           [.] UTF8::convert_to_unicode(char const*, unsigned short*, int)                                ▒
   0.53%  [kernel]            [k] _spin_unlock_irqrestore                                                                    ▒
   0.43%  libjvm.so           [.] UTF8::unicode_length(char const*)                                                          ▒
   0.40%  libjvm.so           [.] java_lang_StackTraceElement::create(Handle, int, int, int, int, Thread*)                   ▒
   0.35%  libjvm.so           [.] BacktraceBuilder::push(Method*, int, Thread*)                                              ▒
   0.34%  libjvm.so           [.] java_lang_Throwable::fill_in_stack_trace(Handle, methodHandle, Thread*)                    ▒
   0.31%  libjvm.so           [.] InstanceKlass::method_with_orig_idnum(int, int)                                            ▒
   0.29%  libjvm.so           [.] UTF8::unicode_length(char const*, int)                                                     ▒
   0.28%  libjvm.so           [.] CollectedHeap::common_mem_allocate_init(KlassHandle, unsigned long, Thread*)               ▒
   0.26%  [kernel]            [k] iowrite16                                                                                  ▒
   0.23%  libjvm.so           [.] Symbol::as_unicode(int&) const                                                             ▒
   0.21%  libpthread-2.12.so  [.] pthread_getspecific                                                                        ▒
   0.20%  libjvm.so           [.] CodeHeap::find_start(void*) const                                                          ▒
   0.20%  perf-31427.map      [.] jlong_disjoint_arraycopy                                                                   ▒
   0.20%  perf-31427.map      [.] jshort_disjoint_arraycopy                                                                  ▒
   0.19%  libjvm.so           [.] resource_allocate_bytes(unsigned long, AllocFailStrategy::AllocFailEnum)                   ▒
   0.19%  libjvm.so           [.] objArrayOopDesc::obj_at(int) const
```

然后，看了具体的代码，发现代码里面有一个Map，这个Map里面存放了每次请求的打点，但是，有地方误用了这个map，导致请求完之后，打点的数据没有清理，结果Map越来越大，每次遍历map也越来越耗CPU。

#### 一点思考

问题的定位比较简单，花了不到半个小时。 但是，事后跟运维聊起来，觉得整个过程还是要吸收一下教训。

CPU偏高的这个问题，在线上已经持续了好几周了，业务的机器扩容了将近三分之一。而如果这个问题不解决，业务还要继续扩容下去。运维中间要求业务排查，但是没有找出问题，因为担心影响其他业务，于是只能先扩容。

程序开发人员平常思考的方向都是快速响应业务，保证接口返回正确；但是，从整个公司从面，一个很小的修改，可能会节省很大的成本（可以降低1/3的机器数），所以，<strong>一个好的程序开发人员，一方面要面向客户优化接口，保证业务的正确和稳定，另一方面，也要积极面向运维优化自己的项目，用最小的成本满足客户的需求。</strong>

