---
title: spring自研框架IoC02实现Bean容器
date: 2020-06-29 16:03:57
tags: spring框架
categories: spring框架篇
urlname: 自研springIoC02
---

### 2.3 实现容器

#### 2.3.1 枚举单例容器

在框架中Bean容器需要保证唯一，适合采用**单例模式**

但是一般的单例模式容易受到**反射和序列化的攻击**，采用**枚举单例**，能有效避免

```java
@Slf4j
@NoArgsConstructor(access = AccessLevel.PRIVATE) //定义一个无参私有构造器
public class BeanContainer {
    //获取单例容器
    public static BeanContainer getInstance(){
        return ContainerHolder.HOLDER.instance;
    }
    private enum ContainerHolder{
        HOLDER;
        private BeanContainer instance;
        ContainerHolder(){
            instance = new BeanContainer();
        }
    }
}
```

#### 2.3.2 实现容器

- 保存Class对象及其实例载体
- 容器的加载：定义并获取目标对象
- 容器的操作方式：对外提供操作

#### 2.3.2.1 保存Class对象及其实例载体

```java
/**
 *  存放所有被配置标记的目标对象的Map
 *  采用ConcurrentHashMap，因为它的并发性能较好
 */
private final Map<Class<?>,Object> beanMap = new ConcurrentHashMap<>();
```



#### 2.3.2.2 容器的加载

1.定义loaded变量 private boolean loaded = false，加载前判断容器是否已经被加载

2.扫描包下的所有class文件：运用包装好的ClassUtil.extractPackageClass(packageName)

3.将获取的classSet进行遍历，判断是否被 **加载bean的注解列表** 所注释，有则利用反射创建实例放入map中

```java
/**
 * 扫描并加载所有的Bean
 * @param packageName 包名
 */
public synchronized void loadBeans(String packageName){
    //判断bean容器是否被加载过
    if(isLoaded()){
        log.warn("Bean Container is already loaded");
        return;
    }
    //提取包下的所有class对象
    Set<Class<?>> classSet = ClassUtil.extractPackageClass(packageName);
    if(classSet==null||classSet.isEmpty()){
        log.warn("extract nothing from package:"+packageName);
        return;
    }
    //遍历classSet
    for(Class<?> clazz:classSet){
        for(Class<? extends Annotation> annotation:BEAN_ANNOTATION){
            //如果类上面标记了上面的注解
            if(clazz.isAnnotationPresent(annotation)){
                //将目标本身作为key，目标类的实例作为值，放入beanMap中
                beanMap.put(clazz,ClassUtil.newInstance(clazz,true));
            }
        }
    }
    loaded = true;
}
```

**加载bean的注解列表：即容器需要扫描那些注解**

```java
/**
 * 加载bean的注解列表:即容器需要扫描那些注解
 */
private static final List<Class<? extends Annotation>> BEAN_ANNOTATION
        = Arrays.asList(Component.class, Controller.class, Service.class, Repository.class);
```

**简单的一个UT：**

```java
private static BeanContainer beanContainer;

//在UT开始之前会进行初始化
@BeforeAll
static void init(){
    beanContainer = BeanContainer.getInstance();
}

@Test
public void loadBeansTest(){
    Assertions.assertEquals(false,beanContainer.isLoaded());
    beanContainer.loadBeans("com.imooc");
    Assertions.assertEquals(6,beanContainer.size());
    Assertions.assertEquals(true,beanContainer.isLoaded());
}
```

#### 2.3.2.3 容器的操作方式

**涉及容器的CRUD**：

- 增加、删除操作
- 根据Class获取对应实例
- 获取所有的Class和实例
- 通过注解来获取被注解标注的Class
- 通过超类获取对应的子类Class
- 获取容器载体保存的Class的数量

##### 1.增加、删除操作

```java
/**
 * 添加Bean
 * @param clazz Class对象
 * @param bean  Class对象对应的Bean实例
 * @return  原有的Bean实例，没有则返回null
 */
public Object addBean(Class<?> clazz,Object bean){
    return beanMap.put(clazz, bean);
}


/**
 * 删除Bean
 * @param clazz 需要删除的Class键值
 * @return 删除的Bean实例，没有则返回null
 */
public Object removeBean(Class<?> clazz){
    return beanMap.remove(clazz);
}
```

##### 2.根据Class获取对应实例

```java
/**
 * 根据Class对象获取Bean实例
 * @param clazz Class对象
 * @return Class对象对应的Bean实例
 */
public Object getBean(Class<?> clazz){
    return beanMap.get(clazz);
}
```

##### 3.获取所有的Class和实例

```java
/**
 * @return bean容器里的所有Class对象
 */
public Set<Class<?>> getClasses(){
    return beanMap.keySet();
}

/**
 * @return bean容器的所有实例对象
 */
public Set<Object> getBeans(){
    return new HashSet<>(beanMap.values());
}
```

##### 4.通过注解来获取被注解标注的Class

```java
/**
 * 通过注解来获取被注解标记过的Class
 * @param annotation 注解
 * @return Class集合
 */
public Set<Class<?>> getClassesByAnnotation(Class<? extends Annotation> annotation){
    //1.获取beanMap的所有class对象
    Set<Class<?>> keySet = getClasses();
    if(keySet==null||keySet.isEmpty()){
        log.warn("keySet is Empty!");
        return null;
    }
    //2.通过注解筛选被注解标记的Class对象，并添加到classSet里
    Set<Class<?>> classSet = new HashSet<>();
    for(Class<?> clazz:keySet){
        //类是否有相关注解标记
        if(clazz.isAnnotationPresent(annotation)){
            classSet.add(clazz);
        }
    }
    return classSet.size()>0?classSet:null;
}
```

##### 5.通过超类获取对应的子类Class

```java
/**
 * 通过接口或者父类获取实现类或者子类的Class集合，不包括其本身
 * @param interfaceOrClass 接口Class或者弗雷Class
 * @return Class集合
 */
public Set<Class<?>> getClassesBySuper(Class<?> interfaceOrClass ){
    //1.获取beanMap的所有class对象
    Set<Class<?>> keySet = getClasses();
    if(keySet==null||keySet.isEmpty()){
        log.warn("keySet is Empty!");
        return null;
    }
    //2.判断keySet里的元素是否是传入的接口或者类的子类，是则添加到classSet里
    Set<Class<?>> classSet = new HashSet<>();
    for(Class<?> clazz:keySet){
        //判断keySet里的元素是否是传入的接口或者类的子类，并排除等于自己本身
        if(interfaceOrClass.isAssignableFrom(clazz)&&!clazz.equals(interfaceOrClass)){
            classSet.add(clazz);
        }
    }
    return classSet.size()>0?classSet:null;
}
```

**补充：class1.isAssignableFrom(class2)函数**

- 如果class1表示的类或接口与class2表示的**类或接口相同**，返回true
- 如果class1表示class2的**超类或者超接口或者实现类**（隔代也可以），返回true

##### 6.获取容器载体保存的Class的数量

```java
/**
 * @return Bean实例的数量
 */
public int size(){
    return beanMap.size();
}
```















