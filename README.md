
## DRouter：支持多进程的组件化方案

[demo下载](https://github.com/Dovar66/DRouter/blob/master/assets/app-debug.apk)

### 框架特点

    * 完美支持多进程，且不需要使用者去bindService或自定义AIDL.
    * 页面路由：支持给Activity定义url，然后通过url跳转到Activity,支持添加拦截器.
    * 跨进程的事件总线.
    * 支持跨进程的API调用.
    * 充分实现模块解耦，页面路由、API调用、事件总线均支持跨模块使用.
    * 基于AOP引导Module的初始化以及页面、拦截器、provider的自动注册.

### 如何配置
1.在BaseModule中添加依赖：

    api 'com.github.Dovar66.DRouter:router-api:0.0.8'

2.在其他需要用到DRouter的组件中添加注解处理器的依赖：

    annotationProcessor 'com.github.Dovar66.DRouter:router-compiler:0.0.8'

    同时在这些组件的defaultConfig中配置注解参数，指定唯一的组件名：

     defaultConfig {
            javaCompileOptions {
                annotationProcessorOptions {
                    arguments = [moduleName: project.getName()]
                }
            }
        }

3.多进程配置：

    * 如果你的项目需要使用多进程广域路由，那么请让你的Application实现 IMultiProcess 接口，广域路由默认是关闭状态，只有实现了该接口才会启用。

    * 在App module的build.gradle文件中，且必须在apply plugin: 'com.android.application'之后引用编译插件RouterPlugin，具体如下：

        apply plugin: 'com.android.application'

        apply plugin: "com.dovar.router.plugin" //必须在apply plugin: 'com.android.application'之后，否则找不到AppExtension

        buildscript {
            repositories {
                google()
                maven {
                    url "https://plugins.gradle.org/m2/"
                }
            }
            dependencies {
                classpath "gradle.plugin.RouterPlugin:plugin:1.1.6"
            }
        }

### 如何使用

#### 在Application.onCreate()中完成初始化

    DRouter.init(app);

#### 页面路由

     添加Path注解,可通过interceptor设置拦截器:

     @Path(path = "/b/main", interceptor = BInterceptor.class)
     public class MainActivity extends AppCompatActivity {

         @Override
         protected void onCreate(Bundle savedInstanceState) {
             super.onCreate(savedInstanceState);
             setContentView(R.layout.module_b_activity_main);
         }
     }

     然后在项目中使用DRouter进行页面跳转:

     DRouter.navigator("/b/main").navigateTo(mContext);

#### 动作路由(API调用)

    创建相应的Provider子类并添加ServiceLoader注解，然后在类中注册Action:

    @ServiceLoader(key = "a")
    public class AProvider extends Provider {
        @Override
        protected void registerActions() {

            registerAction("test1", new Action() {
                @Override
                public RouterResponse invoke(@NonNull Bundle params, Object extra) {
                    Toast.makeText(appContext, "弹个窗", Toast.LENGTH_SHORT).show();
                    return null;
                }
            });

            registerAction("test2", new Action() {
                 @Override
                 public RouterResponse invoke(@NonNull Bundle params, Object extra) {
                    if (extra instanceof Context) {
                       Toast.makeText((Context) extra, params.getString("content"), Toast.LENGTH_SHORT).show();
                    }
                    return null;
                 }
            });
        }
    }

    接下来就可以在项目中使用:

           DRouter.router("a","test1").route();

           DRouter.router("a","test2")
                           .withString("content","也弹个窗")
                           .extra(context)
                           .route();

#### 事件总线

##### 订阅事件

    生命周期感知，不需要手动取消订阅：
    
        DRouter.subscribe(this, ServiceKey.EVENT_A, new EventCallback() {
            @Override
            public void onEvent(Bundle e) {
                Toast.makeText(MainActivity.this, "/b/main/收到事件A", Toast.LENGTH_SHORT).show();
            }
        });
    
    需要手动取消订阅：
    
        Observer<Bundle> mObserver = DRouter.subscribeForever("event_a", new EventCallback() {
            @Override
            public void onEvent(Bundle e) {
                Toast.makeText(MainActivity.this, "/b/main/收到事件A", Toast.LENGTH_SHORT).show();
            }
        });

##### 发布事件(在任意线程)

     Bundle bundle = new Bundle();
     bundle.putString("content", "事件A");
     DRouter.publish(ServiceKey.EVENT_A, bundle);

##### 退订事件(通过subscribeForever()订阅时,需要及时取消订阅)

     DRouter.unsubscribe("event_a", mObserver);

#### 创建组件初始化入口(非必须)

    在组件中创建BaseAppInit的子类，添加Router注解:

        @Module
        public class AInit extends BaseAppInit {

            @Override
            public void onCreate() {
                super.onCreate();
                //与Application.onCreate()的执行时机相同
                //建议在这里完成组件内的初始化工作
            }
        }
        
### 如何渐进式的进行组件化改造？

    完全的组件化拆分并非一两日就能完成，而我们的项目却总会不断有新的需求等待开发，版本迭代工作几乎注定了我们不可能将项目需求暂停来做组件化。
    那么，版本迭代与组件化拆分就需要同步进行，下面是我的建议：
    
    1.开始准备：
    * 新增APP壳工程，建议参考本项目中的app工程.
    * 将你项目当前的application工程作为公共服务层(后面直接用common_service表示)，当然，由于还未开始拆分组件，所以此时它也是最大的业务组件.
    * 新增一个组件(后面用module_search表示)，建议优先选择一个自己最熟悉或相对简单的业务模块着手，比如我自己公司项目的搜索模块，它只有搜索功能且与其他模块交互很少，所以我选择由它开始。
    
    2.建立依赖链：
    壳工程依赖common_service.
    壳工程依赖module_search.
    module_search依赖common_service.(implementation依赖)
    
    3.引入DRouter:
    参考上面的DRouter使用说明.
    
    4.分离公共服务层与基础架构层(非必须)：
    如果之前你的项目没有分离业务层和基础架构层，那么建议你现在将基础架构从common_service中抽离出来.
    
    5.逐步拆分组件：
        拆分第一个组件：
            * 前面我们已经新增了module_search，所以现在要将搜索模块的代码从common_service中抽离并放入搜索组件，此时搜索组件依然可以直接引用公共服务层的代码，但公共服务层则只能通过路由使用搜索功能.
            * 在开发主线做新需求时，新增的资源文件建议最好放到对应的组件工程中，之前的资源文件可以暂时保留在公共服务层，等待后续由各个组件工程认领走，尽可能少的积压在公共服务层。
            * 权限、Android四大组件、特有的第三方库引用等都应该声明在对应的组件Module中，而不应沉入公共服务层，更不允许进入基础框架层。
        拆分第二个组件、第三个...
        (在第一个组件拆分成功并推入市场后，如果反馈良好，那么就可以继续拆分其他的组件了)
    组件拆分粒度取决于你的业务模块划分和组员开发职责划分，建议不要拆分出太多的组件。

### TODO LIST

* 支持跨APP的组件调用。



我的其他项目：[同学，你的系统Toast可能需要修复一下！](https://github.com/Dovar66/DToast)
