---
layout:     post
title:      "MVC,MVP,MVVM"
subtitle:   ""
date:       2021-07-20 12:00:00
author:     "Yaku"
header-img: "img/post-bg-2015.jpg"
tags:
    - Android
    - MVC
    - MVP
    - MVVM
---

# 1.MVC（Model-View-Controller） 

![img](/img/others/mvc.png)

- 模型层(Model) 负责存储、检索、操纵来自数据库或者网络的数据 
- 视图层(View) 用户界面，一般采用XML文件进行界面的描述 
- 控制层(Controller) 业务逻辑处理 



**1.工作原理**

1. 当用户出发事件的时候，view层会发送指令到controller层，自己不执行业务逻辑。
2. Controller执行业务逻辑并且操作Model，但不会直接操作View，可以说它是对View无知的。
3. model层更新完数据然后对视图进行更新，用户得到反馈。



**2.MVC代码实例**

1.先实现一个 model，需要有通知View更新的能力，当model加载成功，模拟从网络或者本地获取数据，需要告知View更新：

```Java
public class MvcModel {
    String mId;
    MvcActivity mActivity;

    public MvcModel (MvcActivity activity) {
        this.mActivity = activity;
    }
    public void loadModel (){
        //模拟从网络或者本地获取数据
        mId = "20170923";
        mActivity.updateUI(this);
    }
}
```

2.View需要发出点击事件，并且传递Controller，同时需要根据Model更新UI：

```Java
public class MvcActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
       //省略
        final MvcController controller = new MvcController(this);
        mFindBtn.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                controller.loadData();
            }
        });
    }

    public void updateUI (MvcModel model) {
        mID.setText(model.mId);
    }
}
```

3.Controller：有时候我们的Activity既充当View也充当Controller，这里为了更好的理解MVC，将Activity进行了拆解：

```Java
public class MvcController {
    MvcActivity mActivity;
    MvcModel mModel;

    MvcController (MvcActivity activity) {
        this.mActivity = activity;
    }

    public void loadData (){
        mModel = new MvcModel(mActivity);
        mModel.loadModel();
    }
}
```



**3.MVC调用链**

View.OnClick -> Controller.loadData -> Model.loadModel -> View.updateUI 



**4.MVC优缺点**

**优点：** 
1. 把业务逻辑全部分离到Controller中，模块化程度高。当业务逻辑变更的时候，不需要变更View和Model，只需要Controller换成另外一个Controller就行了。

 

**缺点：** 

1. Controller测试困难。因为视图同步操作是由View自己执行，而View只能在有UI的环境下运行。在没有UI环境下对Controller进行单元测试的时候，Controller业务逻辑的正确性是无法验证的：Controller更新Model的时候，无法对View的更新操作进行断言。 

2. xml作为view层，控制能力实在太弱，Activity基本上都是View和Controller的合体，既要负责视图的显示又要加入控制逻辑，承担的功能很多，导致代码量很大。如想去动态的改变一个页面的背景，或者动态的隐藏/显示一个按钮，这些都没办法在xml中做，只能把代码写在activity中，造成了activity既是controller层。 

3. view层和model层之间存在耦合。 



# 2.MVP（Model-View-Presenter） 

![img](/img/others/mvp.png)

- 模型层(Model) 负责存储、检索、操纵来自数据库或者网络的数据。 
- 视图层(View) 用户界面，一般采用XML文件进行界面的描述。 
- 逻辑处理层（Presenter） 作为View与Model交互的中间纽带，处理与用户交互的负责逻辑。 



**1.工作原理**

1. View接受用户请求
2. View传递请求给Presenter
3. Presenter做逻辑处理，修改Model
4. Model通知Presenter数据变化
5. Presenter更新View



**2.MVP代码实例**

1.MVP中Model、View、Presenter中的联系件：

```Java
public interface MvpContract {
    interface View {
        void updateUI();
    }

    interface Model {
        void loadModel();
    }

    interface Presenter {
        void loadData ();
    }
}
```

2.还在MVC的例子上变动，需要先对Model进行封装，当loadModel后，不直接通知View更新，而是通知Presenter。

```Java
public class MvpModel implements MvpContract.Model{
    String mId;
    OnGetListener mListener;

    public MvpModel (OnGetListener listener) {
        this.mListener = listener;
    }
   @Override
    public void loadModel (){
        //模拟从网络或者本地获取数据
        mId = "20170923";
        if (mListener != null) {
            mListener.onSuccess(this);
        }
    }
    interface OnGetListener {
        void onSuccess (MvpModel model);
    }
}
```

3.View需要发出点击事件，并且传递给Presenter ，最后也由Presenter去通知View更新UI：

```Plaintext
public class MvpActivity extends AppCompatActivity implements MvpContract.View {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
       //省略
        final MvpPresenter presenter = new MvpPresenter(this);
        mFindBtn.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                presenter.loadData();
            }
        });
    }
    @Override
    public void updateUI (MvpModel model) {
        mID.setText(model.mId);
    }
}
```

4.Presenter，接收到来自View的操作命令后，进行逻辑处理，处理Model，修改完成后 通知View进行修改。

```Plaintext
public class MvpPresenter implements MvpContract.Presenter  {
    MvpContract.View  mView;
    MvpContract.Model mModel;

    MvpPresenter (MvpContract.View view) {
        this.mView= view;
    }
    @Override
    public void loadData (){
        mModel = new MvpModel(new MvpModel.OnGetListener() {
            @Override
            public void onSuccess(MvpModel model) {
                mView.updateUI(model);
            }
        });
        mModel.loadModel ();
    }
}
```



**3.MVP调用链**

View:OnClick -> Presenter:loadData -> Model:loadModel -> Presenter:onSuccess -> View:updateUI 



**4.MVP优缺点**

**优点：**

1. 便于测试。Presenter对View是通过接口进行，在对Presenter进行不依赖UI环境的单元测试的时候。可以通过Mock一个View对象，这个对象只需要实现了View的接口即可。然后依赖注入到Presenter中，单元测试的时候就可以完整的测试Presenter业务逻辑的正确性。

2. 避免了传统开发模式中View和Model耦合的情况，提高了代码可扩展性、组件复用能力、团队协作的效率。



**缺点：**

1. View(Activity)需要持有Presenter的引用，同时，Presenter也需要持有View(Activity)的引用，增加了控制的复杂度；

2. MVC中Activity的代码很臃肿，转移到MVP的Presenter中，同样造成了Presenter在业务逻辑复杂时的代码臃肿。



# 3.MVVM（Model-View-ViewModel） 

![img](/img/others/mvvm.png)

- 模型层(Model) 负责存储、检索、操纵来自数据库或者网络的数据 
- 视图层(View) 用户界面，一般采用XML文件进行界面的描述 
- 视图-模型层(ViewModel) 负责View和Model之间的通信，以此分离视图和数据。 



**1. 工作原理**

1. View接收用户交互请求
2. View将请求转交给ViewModel
3. ViewModel操作Model数据更新
4. Model更新完数据，通知ViewModel数据发生变化
5. ViewModel更新View数据



**2. MVVM代码实例**

1.Model

```Plaintext
public class MvvmModel {
    String mId;
    OnGetListener mListener;

    public MvvmModel(OnGetListener listener) {
        this.mListener = listener;
    }
    public void loadModel() {
        //模拟从网络或者本地获取数据
        mId = "20170923";
        if (mListener != null) {
            mListener.onSuccess(this);
        }
    }
    interface OnGetListener {
        void onSuccess (MvvmModel model);
    }
}
```

2.ViewModel

```Plaintext
public class MvvmViewModel extends BaseObservable {
    public ObservableField<String> mId = new ObservableField<>();
    public MvvmViewModel() {

    }
    public void doGetId(View view) {
        loadData();
    }
    public void loadData() {
        MvvmModel model = new MvvmModel(new MvvmModel.OnGetListener() {
            @Override
            public void onSuccess(MvvmModel model) {
                mId.set(model.mId);
            }
        });
        model.loadModel();
    }
}
```

3.接着使用databinding语法 对 xml 进行数据绑定,我们将 Click事件、输出结果都绑定到ViewModel上。

```Plaintext
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:tools="http://schemas.android.com/tools">
    <data>
        <variable name="viewModel" type="com.liujc.mvvm.MvvmViewModel"/>
    </data>
    <LinearLayout   
        <TextView
                android:id="@+id/id"
                android:layout_width="match_parent"
                android:layout_height="20dp"
                android:text="@{viewModel.mId}"/>
          <Button
                android:id="@+id/findBtn"
                android:onClick="@{viewModel.doGetId}"
                android:text="获取Id"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"/>
    </LinearLayout>
</layout>
```

4.最后在View(Activity)中引入ViewModel：

```Plaintext
public class MvvmActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        ActivityMvvmBinding binding =   DataBindingUtil.setContentView(this,R.layout.activity_mvvm);

        final MvvmViewModel viewModel = new MvvmViewModel();
        binding.setViewModel(viewModel);
    }

}
```



**3.MVVM优缺点**

**优点：**

1. 低耦合。View可以独立于Model变化和修改，一个ViewModel可以绑定到不同的”View”上，当View变化的时候Model可以不变，当Model变化的时候View也可以不变。
2. 可重用性。你可以把一些视图逻辑放在一个ViewModel里面，让很多view重用这段视图逻辑。
3. 独立开发。开发人员可以专注于业务逻辑和数据的开发（ViewModel），设计人员可以专注于页面设计，生成xml代码。
4. ViewModel解决MVP中View(Activity)和Presenter相互持有对方应用的问题，界面由数据进行驱动，响应界面操作无需由View(Activity)传递，数据的变化也无需Presenter调用View(Activity)实现，使得数据传递的过程更加简洁，高效。



**缺点：**

1. ViewModel中存在对Model的依赖。
2. 数据绑定使得 Bug 很难被调试。你看到界面异常了，有可能是你 View 的代码有 Bug，也可能是 Model 的代码有问题。
3. IDE不够完善（修改ViewModel的名称对应的xml文件中不会自动修改等）。



**DataBinding** 

DataBinding是2015年谷歌 I/O大会上介绍了一个数据绑定框架，以前我们可能需要在每个Activity里写很多的findViewById，不仅麻烦，还增加了代码的耦合性，如果我们使用DataBinding，就可以抛弃那么多的findViewById，省时省力。

双向绑定的概念让传统的布局文件由被动转为主动，数据驱动UI，而且View与ViewModel实现了完美的解耦，这也解决了MVP模式下的缺点。 



# 小结 

- 从MVC、MVP到MVVM，实际上是模型和视图的分离过程。MVC中模型和视图没有完全分离，造成Activity代码臃肿，MVP中通过Presenter来进行中转，模型和视图彻底分离，但由于V和P互相引用，代码不够优雅。ViewModel通过Data Binding实现了视图和数据的绑定，解决了这种MVP的缺陷。 
- 可参考一套Android App基础框架 架构设计：从MVC、MVP到MVVM 网络访问：支持REST、HTTPS及SPDY的Retrofit+Okhttp 响应式编程：RxJava/RxAndroid解决方案 依赖注入：Dagger2和ButterKnife使用 
- 框架的选择 任何的项目框架，都是为项目服务的。没有绝对的好坏之分，只有更合适的选择。在项目进展的不同阶段，做出最合适的调整，才是是更适合团队项目发展的框架。谨记任何的项目设计，都是要围绕项目发展阶段，团队成员规模，和团队整体能力而定的。切莫为了设计而设计，为了框架而框架。快速，高效的配合整个团队进展项目，才是最合适的架构。 