@PostConstruct很多人都不会陌生，很多人都会用PostConstruct来进行一些初始化相关的操作，被PostConstruct标注的方法会在对应的Bean完成依赖注入之后执行，所以并不是容器刷新完成后才执行，那么PostConstruct注解到底怎么生效的？接着往下说。

先看个实际的简单的例子，并试着回答一下从Bean实例化开始分别都经历了哪些过程。

```java
@Slf4j
@Component
public class TestPost {

    @Autowired
    private TestDemo testDemo;


    @PostConstruct
    public void testA(Strig args){
        try {
            Thread.sleep(3000);
        }catch (Exception ignore){
        }
        testDemo = new TestDemo();
    }
}
```

- Spring容器首先要将TestDemo给注入进来，如果当前容器内没有TestDemo，那就先去把TestDemo给注册到容器里
- 然后相关的依赖都注入完成了，开始执行被PostConstruct标注着的testA。
- 先休眠个3秒，然后新创建一个TestDemo的实例并指向成员变量testDemo。

然而其实会发生的就是根本执行不到testA的内容就报错了

```text
Lifecycle method annotation requires a no-arg method: public void com.demo.demo.service.TestPost.testA(java.lang.String)
```

你扣了扣脑门，这是为什么啊？你管我有没有入参呢？你Spring又是啥时候偷偷检查的？这个放在后面来讲，我们再进入一个代码并试着分析一下输出顺序。再扩展一下，假如父类也有初始化方法，那么先子类还是先父类的呢？

```java
@Slf4j
@Component
public class TestPost {


    @PostConstruct
    public void testC(){
        log.info("method C run! ");
    }

    @PostConstruct
    public void testA(){
        log.info("method A run!");
    }

    @PostConstruct
    public void testB(){
        log.info("method B run!");
    }
}
```

**接下来就一步一步顺着揭开面纱**

看着朴实无华的标准注解写法。

```java
@Documented
@Retention (RUNTIME)
@Target(METHOD)
public @interface PostConstruct {
}
```

你试着看了看哪里用到了这个注解，好家伙，原来就是CommonAnnotationBeanPostProcessor你这个后置处理器用到了啊。

```java
public class CommonAnnotationBeanPostProcessor extends InitDestroyAnnotationBeanPostProcessor
        implements InstantiationAwareBeanPostProcessor, BeanFactoryAware, Serializable {

    public CommonAnnotationBeanPostProcessor() {
        setOrder(Ordered.LOWEST_PRECEDENCE - 3);
        //真是熟悉的身影
        setInitAnnotationType(PostConstruct.class);
        //猜都猜的到这个小伙子会在这里
        setDestroyAnnotationType(PreDestroy.class);
        ignoreResourceType("javax.xml.ws.WebServiceContext");
    }
}
```

setInitAnnotationType是父类InitDestroyAnnotationBeanPostProcessor的方法之一，主要是就是指定配置bean之后的初始化注解类型。

```java
public class InitDestroyAnnotationBeanPostProcessor  {

    //这里就给你设定一下，传入的就是PostConstruct
    public void setInitAnnotationType(Class<? extends Annotation> initAnnotationType) {
        this.initAnnotationType = initAnnotationType;
    }

    //这里就是用到的地方，clazz是我们对应的bean
    private LifecycleMetadata buildLifecycleMetadata(final Class<?> clazz) {
        if (!AnnotationUtils.isCandidateClass(clazz, Arrays.asList(this.initAnnotationType, this.destroyAnnotationType))) {
            return this.emptyLifecycleMetadata;
        }

        //初始化方法集合
        List<LifecycleElement> initMethods = new ArrayList<>();
        //销毁方法集合
        List<LifecycleElement> destroyMethods = new ArrayList<>();
        //targetClass需要处理的Class
        Class<?> targetClass = clazz;
        do {
            final List<LifecycleElement> currInitMethods = new ArrayList<>();
            final List<LifecycleElement> currDestroyMethods = new ArrayList<>();

            ReflectionUtils.doWithLocalMethods(targetClass, method -> {
                //看看是不是被initAnnotationType也就是我们的PostConstruct注解标注了
                if (this.initAnnotationType != null && method.isAnnotationPresent(this.initAnnotationType)) {
                    //生成LifecycleElement并加到初始化方法列表里，这里就是不然你加方法入参的地方，后面说
                    LifecycleElement element = new LifecycleElement(method);
                    currInitMethods.add(element);
                    if (logger.isTraceEnabled()) {
                        logger.trace("Found init method on class [" + clazz.getName() + "]: " + method);
                    }
                }
                if (this.destroyAnnotationType != null && method.isAnnotationPresent(this.destroyAnnotationType)) {
                    currDestroyMethods.add(new LifecycleElement(method));
                    if (logger.isTraceEnabled()) {
                        logger.trace("Found destroy method on class [" + clazz.getName() + "]: " + method);
                    }
                }
            });
            //由于是从此类往父类开找，这个我们一眼就看出来父类的先执行了
            initMethods.addAll(0, currInitMethods);
            //不言而喻
            destroyMethods.addAll(currDestroyMethods);
            //指向父类
            targetClass = targetClass.getSuperclass();
        }
        //也就是往父类挨个找，直到父类为空或者就是Object类
        while (targetClass != null && targetClass != Object.class);

        return (initMethods.isEmpty() && destroyMethods.isEmpty() ? this.emptyLifecycleMetadata :
                new LifecycleMetadata(clazz, initMethods, destroyMethods));
    }
  
    //就是它做的事儿
    public LifecycleElement(Method method) {
        //你这方法入参要是不为0，直接就是一手报错，没问题吧
        if (method.getParameterCount() != 0) {
            throw new IllegalStateException("Lifecycle method annotation requires a no-arg method: " + method);
        }
        this.method = method;
        this.identifier = (Modifier.isPrivate(method.getModifiers()) ?
                ClassUtils.getQualifiedMethodName(method) : method.getName());
    }
}
```

bean的后置处理器什么时候被调用，自己怎么自定义bean后置处理器这个留在以后再说。

回到开始的问题，一个类中多个PostConstruct执行顺序又怎么说？首先就是我们自己如果跑一下代码应该可以很快得知内容。

第一次的运行结果： A -> B -> C，真相那都就是name的排序结果？

```text
com.demo.demo.service.TestPost           : method A run!
com.demo.demo.service.TestPost           : method B run!
com.demo.demo.service.TestPost           : method C run! 
```

第二次的运行结果：C -> B -> A，这结果真是很量子力学，因为感受到我们的观测，所以第二次程序就按照相反的顺序执行。

```text
com.demo.demo.service.TestPost           : method C run! 
com.demo.demo.service.TestPost           : method B run!
com.demo.demo.service.TestPost           : method A run!
```

第三次的运行结果：B -> C -> A，打扰了，搁这儿玩全排序呢？

```text
com.demo.demo.service.TestPost           : method B run!
com.demo.demo.service.TestPost           : method C run! 
com.demo.demo.service.TestPost           : method A run!
```

三次都是亲手跑的，不是我自己脑补写上去的，实际上这玩意就是随机的，也就是顺序不固定。那问题来了，为什么不固定？Spring写了个随机函数？允许我加粗个标题。

## 多个@PostConstruct标注方法顺序为什么是随机的？

回到我们上面的InitDestroyAnnotationBeanPostProcessor后置处理器的buildLifecycleMetadata方法，我再贴一次

```java
public class InitDestroyAnnotationBeanPostProcessor  {
  
    private LifecycleMetadata buildLifecycleMetadata(final Class<?> clazz) {
        if (!AnnotationUtils.isCandidateClass(clazz, Arrays.asList(this.initAnnotationType, this.destroyAnnotationType))) {
            return this.emptyLifecycleMetadata;
        }
        List<LifecycleElement> initMethods = new ArrayList<>();
        List<LifecycleElement> destroyMethods = new ArrayList<>();
        Class<?> targetClass = clazz;
        do {
            final List<LifecycleElement> currInitMethods = new ArrayList<>();
            final List<LifecycleElement> currDestroyMethods = new ArrayList<>();

            //其实重要的就是这个，调用了ReflectionUtils的doWithLocalMethods
            //doWithLocalMethods的两个入参:一个目标类一个函数式接口MethodCallback并在这层写了doWith实现
            ReflectionUtils.doWithLocalMethods(targetClass, method -> {
                if (this.initAnnotationType != null && method.isAnnotationPresent(this.initAnnotationType)) {
                    LifecycleElement element = new LifecycleElement(method);
                    //这个currInitMethods是个list，所以添加顺序就是固定的，那么底层的遍历顺序才是决定方法顺序的地方
                    //所以需要到ReflectionUtils.doWithLocalMethods去查看原因
                    currInitMethods.add(element);
                    if (logger.isTraceEnabled()) {
                        logger.trace("Found init method on class [" + clazz.getName() + "]: " + method);
                    }
                }
                if (this.destroyAnnotationType != null && method.isAnnotationPresent(this.destroyAnnotationType)) {
                    currDestroyMethods.add(new LifecycleElement(method));
                    if (logger.isTraceEnabled()) {
                        logger.trace("Found destroy method on class [" + clazz.getName() + "]: " + method);
                    }
                }
            });
            initMethods.addAll(0, currInitMethods);
            destroyMethods.addAll(currDestroyMethods);
            targetClass = targetClass.getSuperclass();
        }
        while (targetClass != null && targetClass != Object.class);

        return (initMethods.isEmpty() && destroyMethods.isEmpty() ? this.emptyLifecycleMetadata :
                new LifecycleMetadata(clazz, initMethods, destroyMethods));
    }
}
```

嗖的一下啊，我们就来到了ReflectionUtils.doWithLocalMethods这里

```java
public abstract class ReflectionUtils {
    public static void doWithLocalMethods(Class<?> clazz, MethodCallback mc) {
        //下面是对methods的遍历，所以methods数组的顺序就决定了后面执行的顺序
        Method[] methods = getDeclaredMethods(clazz, false);
        for (Method method : methods) {
            try {
                mc.doWith(method);
            }
            catch (IllegalAccessException ex) {
                throw new IllegalStateException("Not allowed to access method '" + method.getName() + "': " + ex);
            }
        }
    }

    private static Method[] getDeclaredMethods(Class<?> clazz, boolean defensive) {
        Assert.notNull(clazz, "Class must not be null");
        //这里有个declaredMethodsCache，正常的缓存设计，那我们肯定看没有缓存从源头拿的时候了
        Method[] result = declaredMethodsCache.get(clazz);
        if (result == null) {
            try {
                //这里就是核心了，这里就跳到Class类去了，想来也是肯定的，毕竟要获取类的信息
                Method[] declaredMethods = clazz.getDeclaredMethods();
                //这个是找接口层默认实现的方法，甭管它
                List<Method> defaultMethods = findConcreteMethodsOnInterfaces(clazz);
                if (defaultMethods != null) {
                    result = new Method[declaredMethods.length + defaultMethods.size()];
                    System.arraycopy(declaredMethods, 0, result, 0, declaredMethods.length);
                    int index = declaredMethods.length;
                    for (Method defaultMethod : defaultMethods) {
                        result[index] = defaultMethod;
                        index++;
                    }
                }
                else {
                    result = declaredMethods;
                }
                declaredMethodsCache.put(clazz, (result.length == 0 ? EMPTY_METHOD_ARRAY : result));
            }
            catch (Throwable ex) {
                throw new IllegalStateException("Failed to introspect Class [" + clazz.getName() +
                        "] from ClassLoader [" + clazz.getClassLoader() + "]", ex);
            }
        }
        return (result.length == 0 || !defensive) ? result : result.clone();
    }

}
```

让我们来到JAVA领域的核心类，Class类

```java
public final class Class<T> implements java.io.Serializable, GenericDeclaration, Type, AnnotatedElement {
  
    @CallerSensitive
    public Method[] getDeclaredMethods() throws SecurityException {
        //检查行为
        checkMemberAccess(Member.DECLARED, Reflection.getCallerClass(), true);
        //好了，就是privateGetDeclaredMethods方法来的，快要接近终点了，马上就要真相大白了
        return copyMethods(privateGetDeclaredMethods(false));
    }

    private Method[] privateGetDeclaredMethods(boolean publicOnly) {
        //前面的我得懒得管
        checkInitted();
        Method[] res;
        //老规矩我不看你的缓存的是啥
        ReflectionData<T> rd = reflectionData();
        if (rd != null) {
            res = publicOnly ? rd.declaredPublicMethods : rd.declaredMethods;
            if (res != null) return res;
        }
        // 拿不到之后的操作才是源头操作，很明显不出意外啊getDeclaredMethods0就是最终的获取方法，顺序问题重要接近尾声了，点进去就是结果了
        res = Reflection.filterMethods(this, getDeclaredMethods0(publicOnly));
        if (rd != null) {
            if (publicOnly) {
                rd.declaredPublicMethods = res;
            } else {
                rd.declaredMethods = res;
            }
        }
        return res;
    }

    //居然是个native方法？裂开了啊
    private native Method[]      getDeclaredMethods0(boolean publicOnly);
  
}
```

想了想不能白分析啊，这就打开JDK源码，检索一下我们的getDeclaredMethods0，然后我们在Class.c文件找到了它

```text
static JNINativeMethod methods[] = {
    {"getDeclaredMethods0","(Z)[" MHD,      (void *)&JVM_GetClassDeclaredMethods},
}
```

接下来就是找JVM_GetClassDeclaredMethods了，在jvm.h上对应着有

```text
JNIEXPORT jobjectArray JNICALL
JVM_GetClassDeclaredMethods(JNIEnv *env, jclass ofClass, jboolean publicOnly);

```

在jvm.cpp上对应着有，看来又要去找get_class_declared_methods_helper了

```text
JVM_ENTRY(jobjectArray, JVM_GetClassDeclaredMethods(JNIEnv *env, jclass ofClass, jboolean publicOnly))
{
  return get_class_declared_methods_helper(env, ofClass, publicOnly,
                                           /*want_constructor*/ false,
                                           vmClasses::reflect_Method_klass(), THREAD);
}
JVM_END
```

在get_class_declared_methods_helper，贴一下代码

```text
static jobjectArray get_class_declared_methods_helper(
                                  JNIEnv *env,
                                  jclass ofClass, jboolean publicOnly,
                                  bool want_constructor,
                                  Klass* klass, TRAPS) {

  JvmtiVMObjectAllocEventCollector oam;

  oop ofMirror = JNIHandles::resolve_non_null(ofClass);
  // Exclude primitive types and array types
  if (java_lang_Class::is_primitive(ofMirror)
      || java_lang_Class::as_Klass(ofMirror)->is_array_klass()) {
    // Return empty array
    oop res = oopFactory::new_objArray(klass, 0, CHECK_NULL);
    return (jobjectArray) JNIHandles::make_local(THREAD, res);
  }

  //结合下面的k->methods()我们知道，关键点在于InstanceKlass
  InstanceKlass* k = InstanceKlass::cast(java_lang_Class::as_Klass(ofMirror));

  // Ensure class is linked
  k->link_class(CHECK_NULL);

  //一眼就看到了我们熟悉的东西，methods，也就是其实找的就是k，咱们的instanceKlass的结构变量methods，问题不大
  Array<Method*>* methods = k->methods();
  int methods_length = methods->length();

  // Save original method_idnum in case of redefinition, which can change
  // the idnum of obsolete methods.  The new method will have the same idnum
  // but if we refresh the methods array, the counts will be wrong.
  ResourceMark rm(THREAD);
  GrowableArray<int>* idnums = new GrowableArray<int>(methods_length);
  int num_methods = 0;

  for (int i = 0; i < methods_length; i++) {
    methodHandle method(THREAD, methods->at(i));
    if (select_method(method, want_constructor)) {
      if (!publicOnly || method->is_public()) {
        idnums->push(method->method_idnum());
        ++num_methods;
      }
    }
  }

  // Allocate result
  objArrayOop r = oopFactory::new_objArray(klass, num_methods, CHECK_NULL);
  objArrayHandle result (THREAD, r);

  // Now just put the methods that we selected above, but go by their idnum
  // in case of redefinition.  The methods can be redefined at any safepoint,
  // so above when allocating the oop array and below when creating reflect
  // objects.
  for (int i = 0; i < num_methods; i++) {
    methodHandle method(THREAD, k->method_with_idnum(idnums->at(i)));
    if (method.is_null()) {
      // Method may have been deleted and seems this API can handle null
      // Otherwise should probably put a method that throws NSME
      result->obj_at_put(i, NULL);
    } else {
      oop m;
      if (want_constructor) {
        m = Reflection::new_constructor(method, CHECK_NULL);
      } else {
        m = Reflection::new_method(method, false, CHECK_NULL);
      }
      result->obj_at_put(i, m);
    }
  }

  return (jobjectArray) JNIHandles::make_local(THREAD, result());
}
```

找到instanceKlass.hpp，让我们直接找到methods()

```text
  // 在这里
  Array<Method*>* methods() const          { return _methods; }
  // 那必然是set干的顺序的事儿了啊
  void set_methods(Array<Method*>* a)      { _methods = a; }
  Method* method_with_idnum(int idnum);
  Method* method_with_orig_idnum(int idnum);
  Method* method_with_orig_idnum(int idnum, int version);
```

通过set_methods我们找到了classFileParser.cpp，在类加载的时候进行了调用

```text
void ClassFileParser::apply_parsed_class_metadata(
                                            InstanceKlass* this_klass,
                                            int java_fields_count) {
  assert(this_klass != NULL, "invariant");

  _cp->set_pool_holder(this_klass);
  this_klass->set_constants(_cp);
  this_klass->set_fields(_fields, java_fields_count);
  //就是这里
  this_klass->set_methods(_methods);
  
  //而_methods又经历了一次sort
  _method_ordering = sort_methods(_methods);
  
  
  //排一排
  static const intArray* sort_methods(Array<Method*>* methods) {
  const int length = methods->length();
  if (JvmtiExport::can_maintain_original_method_order() || Arguments::is_dumping_archive()) {
    for (int index = 0; index < length; index++) {
      Method* const m = methods->at(index);
      assert(!m->valid_vtable_index(), "vtable index should not be set");
      m->set_vtable_index(index);
    }
  }
  
  //又再调用了Method的sort_methods
  Method::sort_methods(methods);

  intArray* method_ordering = NULL;
  if (JvmtiExport::can_maintain_original_method_order() || Arguments::is_dumping_archive()) {
    method_ordering = new intArray(length, length, -1);
    for (int index = 0; index < length; index++) {
      Method* const m = methods->at(index);
      const int old_index = m->vtable_index();
      assert(old_index >= 0 && old_index < length, "invalid method index");
      method_ordering->at_put(index, old_index);
      m->set_vtable_index(Method::invalid_vtable_index);
    }
  }
  return method_ordering;
}
```

于是我们跟随脚步来到了method.cpp

```text
void Method::sort_methods(Array<Method*>* methods, bool set_idnums, method_comparator_func func) {
  int length = methods->length();
  if (length > 1) {
    if (func == NULL) {
      //就这个传递进来的方法排序比较器，找一找
      func = method_comparator;
    }
    {
      NoSafepointVerifier nsv;
      QuickSort::sort(methods->data(), length, func, /*idempotent=*/false);
    }
    // Reset method ordering
    if (set_idnums) {
      for (int i = 0; i < length; i++) {
        Method* m = methods->at(i);
        m->set_method_idnum(i);
        m->set_orig_method_idnum(i);
      }
    }
  }
}


static int method_comparator(Method* a, Method* b) {
  //啥？为啥是按照名字排的啊？那。。。。肯定不是按照名字来了
  return a->name()->fast_compare(b->name());
}
```

在method.cpp里我们可以看到#include "oops/constMethod.hpp"，对应的在constMethod.hpp中，原来你name是个Symbol啊

```text
  Symbol* name() const                           { return constants()->symbol_at(name_index()); }
```

而在Symbol.hpp中我们可以找到fast_compare，看到注释我们也知道了为了速度，采用的是Symbol的地址来比较的，每次启动在C_HEAP分配时这个地址先后顺序就没法保证了。

```text
// Note: this comparison is used for vtable sorting only; it doesn't matter
// what order it defines, as long as it is a total, time-invariant order
// Since Symbol*s are in C_HEAP, their relative order in memory never changes,
// so use address comparison for speed
int Symbol::fast_compare(const Symbol* other) const {
 return (((uintptr_t)this < (uintptr_t)other) ? -1
   : ((uintptr_t)this == (uintptr_t) other) ? 0 : 1);
}
```
