最近，看到一些小伙伴想要入门Dagger2，加之最近刚经历了Dagger2的水深火热，在这里针对Dagger2中不同的注解方式，会生成怎样的代码，结合其生成的不同代码，来帮助大家做一些深入的理解。

<!-- more -->
#概念
首先，Dagger2是一个DI的解决方案，跟之前接触过的Spring相比，它最主要的好处是通过apt插件在编译阶段时，用来生成注入的代码；而Spring是需要在运行时，通过XML或者注解来进行代码注入的。所以，相比来说，在性能上，Dagger2是优于Spring，但带来的是编译阶段时间的延长。这样的话，每当我们修改或者添加这些注解代码的时候，就需要我们重新Build一下（即由apt插件来生成我们所需要使用的代码），build时间的延长，感觉这对Android开发程序员来说，应该习以为常了吧。（掩面而泣。。。）

另外，不得不提的就是apt插件的成熟。apt插件通过生成代码的方式会使得我们则针对特定规则的代码，通过添加注解，使用apt在编译阶段生成代码，减少我们的代码书写量。在gayhub上，已经有很多成熟的库，在使用apt来生成代码，像[bundler](https://github.com/workarounds/bundler)这个库，就是通过封装Intent参数跟界面组件来绑定，这样我们就可不必通过getIntent来一个个获取参数。另外还提供了使用`saveState`和`restoreState`，使得我们在界面组件异常退出的时候，不必再使用`savedInstance`来进行数据的保存与获取。还有就是支持`Parcelable`数据以及自定义数据parser，另作者不由不喜欢啊。

#使用
根项目添加apt的版本依赖

``` java
dependencies {
    ...
   classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'
    ...
}
```
项目中配置apt插件的使用，以及Dagger2的版本

``` java
apply plugin: 'com.neenbedankt.android-apt'

dependencies {
   compile 'com.google.dagger:dagger:2.0.2'
   compile 'com.google.dagger:dagger-compiler:2.0.2'
   compile 'org.glassfish:javax.annotation:10.0-b28'
}
```


#重要概念
这里简单提一下它们的主要作用，dagger2两个很重要的东西，不过介绍的文章很多，需要的话，可自行查找一下。


+ ### Component 
  用来为“被注入方“提供其所需要的注入类。当在子Scoped的Component中，也要被注入相同的类的时候，则需要在当前的Component添加相应的返回对应的类的方法。
  
+ ### Module
  用来提供上层所需要的依赖类。这里所带来的好处就是，所有的依赖类都是通过Module来提供出去，当我们的依赖发生改变时，我们只需要在这里改变提供一个新的对象类即可，而不影响我们上层使用的代码。这里，举一个最简单的例子就是，当我们调用服务器提供的API的时候，我们为上层提供了一个Repository命名的接口，并对其返回的是调用API的类；这里，当API还未实现，我们仅仅对其提供一个MockAPI即可，上层的调用者根本不需要关心我的数据提供从哪里来，是否真实。

# 依赖实现辅助类

###Provider
`Provider`定义了一个来提供泛型T的方式，是个很简单的接口。

``` java
public interface Provider<T> {

   /**
    * Provides a fully-constructed and injected instance of {@code T}.
    *
    * @throws RuntimeException if the injector encounters an error while
    *  providing an instance. For example, if an injectable member on
    *  {@code T} throws an exception, the injector may wrap the exception
    *  and throw it to the caller of {@code get()}. Callers should not try
    *  to handle such exceptions as the behavior may vary across injector
    *  implementations and even different configurations of the same injector.
    */
   T get();
}
```

### ScopedProvider
而`ScopedProvider`在`Provider`的基础上，加上了Scoped的概念，主要通过指定类型的Factory来创建出来对应类型的ScopedProvider，在get方法上，主要通过`double-checked`来确保获取实例T的唯一性。

``` java
/**
 * A {@link Provider} implementation that memoizes the result of a {@link Factory} instance.
 *
 * @author Gregory Kick
 * @since 2.0
 */
public final class ScopedProvider<T> implements Provider<T> {
   private static final Object UNINITIALIZED = new Object();

   private final Factory<T> factory;
   private volatile Object instance = UNINITIALIZED;

   private ScopedProvider(Factory<T> factory) {
      assert factory != null;
      this.factory = factory;
   }

   @SuppressWarnings("unchecked") // cast only happens when result comes from the factory
      @Override
      public T get() {
         // double-check idiom from EJ2: Item 71
         Object result = instance;
         if (result == UNINITIALIZED) {
            synchronized (this) {
               result = instance;
               if (result == UNINITIALIZED) {
                  instance = result = factory.get();
               }
            }
         }
         return (T) result;
      }

   /** Returns a new scoped provider for the given factory. */
   public static <T> Provider<T> create(Factory<T> factory) {
      if (factory == null) {
         throw new NullPointerException();
      }
      return new ScopedProvider<T>(factory);
   }
}
```

###Factory
`Factory`仅仅继承了一个Provider，并是一个空的实现。


``` java
public interface Factory<T> extends Provider<T> {}
```

###MembersInjector
从类的注释中，可以看出它的主要作用是给类的属性字段，或者方法参数来提供注入，并忽略是否在含有构造器的注入，都会生成`MembersInjector`的类。

``` java
/**
 * Injects dependencies into the fields and methods on instances of type {@code T}. Ignores the
 * presence or absence of an injectable constructor.
 *
 * @param <T> type to inject members of
 *
 * @author Bob Lee
 * @author Jesse Wilson
 * @since 2.0 (since 1.0 without the provision that {@link #injectMembers} cannot accept
 *      {@code null})
 */
public interface MembersInjector<T> {

   /**
    * Injects dependencies into the fields and methods of {@code instance}. Ignores the presence or
    * absence of an injectable constructor.
    *
    * <p>Whenever the object graph creates an instance, it performs this injection automatically
    * (after first performing constructor injection), so if you're able to let the object graph
    * create all your objects for you, you'll never need to use this method.
    *
    * @param instance into which members are to be injected
    * @throws NullPointerException if {@code instance} is {@code null}
    */
   void injectMembers(T instance);
}
```


# 注入方式
这里通过列举几种不同的注入方式，探究其的实现，了解它们的不同使用场合

###构造器注入
这里先看一个简单的类

``` java
public class TestData {

   @Inject
   public TestData() {
   }
}
```
通过在构造器上添加`@Inject`注解，则会编译生成一个实现了`Factory`接口的单例类，用来生成TestData类。生成的代码如下：

``` java
@Generated("dagger.internal.codegen.ComponentProcessor")
public enum TestData_Factory implements Factory<TestData> {
   INSTANCE;

   @Override
   public TestData get() {  
      return new TestData()
   }

   public static Factory<TestData> create() {  
      return INSTANCE;
   }
}
```
可以看出通过构造器注入的生成的Factory类，是一个通过enum来实现的一个单例工厂类，是不是有个新技能Get。

### Module注入
实现一个Module来提供TestData的依赖，代码如下：

``` java
@Module
public class TestModule {

   @Singleton
   @Provides
   public TestData provideTestData(){
      return new TestData();
   }
}
```
再查看一下，通过apt生成之后的代码：

``` java
@Generated("dagger.internal.codegen.ComponentProcessor")
public final class TestModule_ProvideTestDataFactory implements Factory<TestData> {
   private final TestModule module;

   public TestModule_ProvideTestDataFactory(TestModule module) {  
      assert module != null;
      this.module = module;
   }

   @Override
      public TestData get() {  
         TestData provided = module.provideTestData();
         if (provided == null) {
            throw new NullPointerException("Cannot return null from a non-@Nullable @Provides method");
         }
         return provided;
      }

   public static Factory<TestData> create(TestModule module) {  
      return new TestModule_ProvideTestDataFactory(module);
   }
}

```
可以看出，针对Module注解，dagger2会根据咱们定义了`Provides`的注解方法，会生成相应以Module为开头的Factory类，而这个Factory会以`Module`作为参数，通过调用Module中的代码，来实现注入类的提供。

### 属性注入，方法注入
这里，以我们常用的`Activity`为例，毕竟`Activity`的使用可是不允许我们通过构造器来生成一个Activity的，此时就是`MembersInjector`的用武之地了。
来看个Activity的代码先。

``` java
public class MainActivity extends AppCompatActivity {

   @Inject
   TestData mData;

   @Override
   protected void onCreate(Bundle savedInstanceState) {
      super.onCreate(savedInstanceState);
      setContentView(R.layout.activity_main);
   }

}
```
当然，也可以使用方法来进行注入，像下面这样：

``` java
@Inject
public void setTestData(TestData testData){
}
```

会生成一个`MainActivity_MembersInjector`的类，如下：

``` java
@Generated("dagger.internal.codegen.ComponentProcessor")
public final class MainActivity_MembersInjector implements MembersInjector<MainActivity> {
   private final MembersInjector<AppCompatActivity> supertypeInjector;
   private final Provider<TestData> mDataProvider;

   public MainActivity_MembersInjector(MembersInjector<AppCompatActivity> supertypeInjector, Provider<TestData> mDataProvider) {  
      assert supertypeInjector != null;
      this.supertypeInjector = supertypeInjector;
      assert mDataProvider != null;
      this.mDataProvider = mDataProvider;
   }

   @Override
      public void injectMembers(MainActivity instance) {  
         if (instance == null) {
            throw new NullPointerException("Cannot inject members into a null reference");
         }
         supertypeInjector.injectMembers(instance);
         instance.mData = mDataProvider.get();
      }

   public static MembersInjector<MainActivity> create(MembersInjector<AppCompatActivity> supertypeInjector, Provider<TestData> mDataProvider) {  
      return new MainActivity_MembersInjector(supertypeInjector, mDataProvider);
   }
}

```
从上方可以看出，`MainActivity_MembersInjector`类通过实现`MembersInjector`类，通过`injectMembers`方法，来给`MainActivity`的实例，采用Provider获取实例的方式，来给我们之前定义的`Inject`参数或者属性，来赋值相应的注入类。
另外，我们看到，在`injectMembers`的方法中，也会调用父类的`supertypepInjector`的类，来调用父类的参数注入。

# Component完成注入
上面，我们提到了Activity所需要的类注入，只能通过方法参数，或者字段属性两个方式，（构造器是行不通的）。其`MembersInjector`的类页编写好了，我们该怎么调用呢？继续上面的例子，我们来给MainActivity来提供注入方式，神奇的Component就登上舞台了。

``` java
@Singleton
@Component(modules = { TestModule.class })
public interface ApplicationComponent {
   void inject(MainActivity mainActivity);

}
```
这里，component一般在使用的时候，都是需要跟Scope相绑定的，不然会报错。看它的生成代码：

``` java
@Generated("dagger.internal.codegen.ComponentProcessor")
public final class DaggerApplicationComponent implements ApplicationComponent {
   private Provider<TestData> provideTestDataProvider;
   private MembersInjector<MainActivity> mainActivityMembersInjector;

   private DaggerApplicationComponent(Builder builder) {  
      assert builder != null;
      initialize(builder);
   }

   public static Builder builder() {  
      return new Builder();
   }

   public static ApplicationComponent create() {  
      return builder().build();
   }

   private void initialize(final Builder builder) {  
      this.provideTestDataProvider = ScopedProvider.create(TestModule_ProvideTestDataFactory.create(builder.testModule));
      this.mainActivityMembersInjector = MainActivity_MembersInjector.create((MembersInjector) MembersInjectors.noOp(), provideTestDataProvider);
   }

   @Override
      public void inject(MainActivity mainActivity) {  
         mainActivityMembersInjector.injectMembers(mainActivity);
      }

   public static final class Builder {
      private TestModule testModule;

      private Builder() {  
      }

      public ApplicationComponent build() {  
         if (testModule == null) {
            this.testModule = new TestModule();
         }
         return new DaggerApplicationComponent(this);
      }

      public Builder testModule(TestModule testModule) {  
         if (testModule == null) {
            throw new NullPointerException("testModule");
         }
         this.testModule = testModule;
         return this;
      }
   }
}

```

我们定义好的`Component`接口，都会生成一个以Dagger开头的一个实现类。在其中，就可以看到它在实现咱们定义的`void inject(MainActivity activity)`方法，就是通过使用`MainActivity_MembersInjector`类，来完成这一步注入的，有木有颇感神奇啊。这就是apt插件的神奇之处，定义好了规则，按照规则来生成代码即可。

这里，紧接着提及一下`Scope`的的用法。当我们为`Component`定义了`Scope`之后，并且在`module`的方法上，添加了`Scope`注解，这样Dagger2在`Component`中生成相应的`Provider`的时候，就会在前面添加`ScopedProvider`，来对我们所需要的`Provider`来提供单例模式的访问。具体可以看到的代码就是上面初始化过程中，生成`provideTestDataProvider`的字段。

# 总结
掌握了上面提及的这些内容，则Dagger2的不同注入方法，生成怎样的代码，我们就上手Dagger2的使用，若是在使用中还遇到问题的话，可加入下方的二维码群，来进行交流讨论。另外，感谢明道小伙伴们([月半兄](https://github.com/yueban)，[jager](https://github.com/laobie)，[lchad](https://github.com/lchad))对Dagger2的讨论。



![Paste_Image.png](http://7xpyth.com1.z0.glb.clouddn.com/android-qq.png)


> PS: 转载，请注明原文链接：http://alighters.com/blog/2016/04/15/dagger2-indepth-understanding/


