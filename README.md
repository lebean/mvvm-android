# MVVM：Android MVVM模式之DataBinding
***
> Google的DataBinding发布已经很长时间了，现在也已经很成熟也比较稳定了。我之前的项目一直使用MVP，其实也一直想换到MVVM模式，毕竟它使用数据驱动，能解决MVP模式中的一些缺陷，但是一直没有太多时间，今天就整理一下Android的MVVM模式怎么用，本文章偏向实用，什么MVVM的概念、优缺点这些就不介绍了，大家各自到各论坛中看看吧，文章很多很多的。



###  启用DataBinding功能

​	在android sudio中新建一个项目，在Module的build.gradle文件android节点中添加databinding{enable true}启用DataBinding，配置好的build.gradle文件内容如下：

```
apply plugin: 'com.android.application'
android {
    compileSdkVersion 25
    buildToolsVersion "25.0.2"
    defaultConfig {
        applicationId "leix.lebean.android.mvvm"
        minSdkVersion 16
        targetSdkVersion 25
        versionCode 1
        versionName "1.0"
        testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"
    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
    dataBinding{
        enabled true
    }
}
dependencies {
    compile fileTree(include: ['*.jar'], dir: 'libs')
    androidTestCompile('com.android.support.test.espresso:espresso-core:2.2.2', {
        exclude group: 'com.android.support', module: 'support-annotations'
    })
    compile 'com.android.support:appcompat-v7:25.3.0'
    compile 'com.android.support.constraint:constraint-layout:1.0.2'
    testCompile 'junit:junit:4.12'
    compile project(':lib')
}
```


## MVVM组成部分

在我的项目中，MVVM由model、view、viewmodel三部分组成

- model：数据模型，一方面存放业务实体的数据，另一方面也负责数据的获取、更新等操作。
- view：视图，视图只负责数据展示，在这里把视图上的事件处理也从view中剥离出来了，事件处理可以独立写在一个EventHandle里，也可以定义到ViewModel中。
- viewmodel：持有view 和 model，负责业务逻辑处理、model与view的协作，而实际上model数据更新后，viewmodel并不需要做什么动作去更新view，这一点与mvp有很大区别，model更新后通过databinding框架直接更新到view了。
- binding：其实除了以上三部分之外，还有一部分内容很重要，只是这部分内容是框架帮我们自动生成的，它就是框架生成的各种ViewDataBinding，在这里我简称它为binding，解决view层数据的绑定问题。



## 把MVVM用起来吧！

在这里讲一个最简单的应用，通过框架绑定数据到view层

1. 新建一个数据模型Author，代码如下
```
/**
 * Name:Author
 * Description:
 * Author:leix
 * Time: 2017/4/5 14:57
 */
public class Author extends BaseObservable {
    private String name;
    private Date birthdate;
    private boolean male;

    @Bindable
    public String getName() {return name;}
    public void setName(String name) {this.name = name;}
    
    @Bindable
    public Date getBirthdate() {return birthdate;}
    public void setBirthdate(Date birthdate) {this.birthdate = birthdate;}

    @Bindable
    public boolean isMale() { return male;}
    public void setMale(boolean male) {this.male = male;}
```
2. 新建一activity的布局文件，在layout节点中添加data定义，Rebuild Project，框架将自动生成AuthorBinding类，
```
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android">
    <data class="AuthorBinding"></data>
    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical"></LinearLayout>
</layout>
```
3. 新建AuthorViewModel，并在xml文件中设置属性绑定
   AuthorViewModel

```
/**
 * Name:AuthorViewModel
 * Description:
 * Author:leix
 * Time: 2017/4/5 15:01
 */
public class AuthorViewModel {
    private Activity activity;
    private ViewDataBinding binding;
    public Author author;
    public AuthorViewModel(Activity activity, ViewDataBinding binding) {
        author = new Author();
        author.setBirthdate(new Date());
        author.setMale(true);
        author.setName("Jim");
        this.activity = activity;
        this.binding = binding;
        this.binding.setVariable(BR.vMode,this);
    }
}
```
xml:
```
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android">
    <data class="AuthorBinding">
        <variable name="vMode" type="cn.com.ssii.mvvm.book.viewmodel.AuthorViewModel" />
    </data>
    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical">
        <TextView
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="@{vMode.author.name}" />
    </LinearLayout>
</layout>
```
activity:
```
public class AuthorActivity extends AppCompatActivity {
    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        ViewDataBinding binding = DataBindingUtil.setContentView(this, R.layout.activity_author);
        AuthorViewModel viewModel = new AuthorViewModel(this, binding);
    }
}
```



### 事件绑定

在使用mvvm后，不用在Activity中去findview，然后设置各的事件了，可能把事件定义到ViewMode或其它类中，在xml文件中定义对应变量并给事件绑定处理方法即可，如：
xml：

```
  <Button
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:onClick="@{vMode.buttonClick}" />
```
viewmodel:

```
public void buttonClick(View view) {
        Toast.makeText(activity, "你点了我一下", Toast.LENGTH_SHORT).show();
    }
```
注意的是这里定义的方法参数要与传统结构相一致，不然会出错误。



### Fragment，和自定义View的使用

Fragment没有setContent方法，还有些情况为了适应平板和手机的需要我们会把一些业务抽象到一个View中，根据设备的大小再把View拼装到界面上来，所以我们的MVVM在View层还要支持View特别是继承ViewGroup然后添加一个从layout中inflate出来的一个View，这两种情况也同样可使用，方法如下
1. fragment
   在fragment中，定义xml文件、model、modelview与之前说的内容是一样的，唯不同可能在实例化databinding的方式不一样，在fragment中我们通过下面语句来实例化一个databinding
```
/**
 * Name:AuthorFragment
 * Description:
 * Author:leix
 * Time: 2017/4/5 16:01
 */
public class AuthorFragment extends Fragment {
    @Nullable
    @Override
    public View onCreateView(LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        ViewDataBinding binding = DataBindingUtil.inflate(inflater, R.layout.activity_author, container, false);
        AuthorViewModel viewModel=new AuthorViewModel(getActivity(),binding);
        return binding.getRoot();
    }
}
```

2. view定义xml文件、model、modelview与之前讲的内容一致，在view的构造函数中调用以下方法来实例化databinding
```
   /**
    * Name:AuthorView
    * Description:
    * Author:leix
    * Time: 2017/4/5 16:08
    */
   public class AuthorView extends FrameLayout {
       public AuthorView(@NonNull Context context) {
           this(context, null);
       }
   	public AuthorView(@NonNull Context context, @Nullable AttributeSet attrs) {
           this(context, null, -1);
       }

       public AuthorView(@NonNull Context context, @Nullable AttributeSet attrs, @AttrRes int defStyleAttr) {
           super(context, attrs, defStyleAttr);
           ViewDataBinding binding = DataBindingUtil.inflate(LayoutInflater.from(getContext()), R.layout.activity_author, this, true);
           AuthorViewModel viewModel = new AuthorViewModel(context, binding);
       }
   }
   
```



## 属性双向绑定

@=

​	在属性绑定的时候使用@=的方式来双向绑定，使用了双向绑定，在EditText中输入内容，就会自动存储到对应的字段上，如果还有其它View单向绑定了这个值，它的显示也会发生变化

```
<EditText
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="@={vModel.book.name}" />
```



## Observable

​	一个纯的Java Model类在被修改后，UI并不会更新。如果我们想数据绑定后修改数据模型的值就可以更新UI怎么破呢？这里就要用到Obsservable了，当前还要配合@Bindable与notifyPropertyChanged一起使用，如下面这个数据模型被绑定到UI后，修改它的值，UI会自动更新。

```
/**
 * Name:Author
 * Description:
 * Author:leix
 * Time: 2017/4/5 14:57
 */
public class Author extends BaseObservable {
    private String name;
    private Date birthdate;
    private boolean male;

    @Bindable
    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
        notifyPropertyChanged(BR.name);
    }

    @Bindable
    public Date getBirthdate() {
        return birthdate;
    }
    public void setBirthdate(Date birthdate) {
        this.birthdate = birthdate;
        notifyPropertyChanged(BR.birthdate);
    }

    @Bindable
    public boolean isMale() {
        return male;
    }
    public void setMale(boolean male) {
        this.male = male;
        notifyPropertyChanged(BR.male);
    }
}
```



## ObservableField

​	如果一个Model里的的有字段都要示修改后自动更新UI，我们还向上面使用Obsservable肯定会产生更多代码，谁也不原写那么多代码是吧？那么我们只要把字段定义为ObservableField就OK了。ObservableField提供的类型有ObservableFile<T>, ObservableBoolean, ObservableByte, ObservableInt, ObservableChar,  ObservableLong, ObservableFloat, ObserbableDouble, ObservableParcelable, 这些基本已经能满足所有数据类型的需要了。

```
/**
 * Name:Book
 * Description:
 * Author:leix
 * Time: 2017/4/5 10:32
 */
public class Book {
    public final ObservableField<String> name = new ObservableField<>();
    public final ObservableField<String> description = new ObservableField<>();
    public final ObservableInt price = new ObservableInt();
    public final ObservableField<Date> publishDate = new ObservableField<>();
}
```

​	在xml中的绑定与之前的字段绑定一样，只是在java中修改值和获取有些不同，java中访问如下：

```
book.name.get();//获取name属性值
book.name.set("Thinking in java");//设置name属性值
```



## 自定义属性

​	之前要给View添加自定义属性总是很麻烦，要去style里定义，再到xml中使用，在view的构造中解析，使用Databinding来实现自定义属性就方便多了，在xml文件中直接设置属性，框架会根据全名空间去查找对应的set方法，但是注意如果没有找到对应set方法就会出错，例如下面我重写一个View 继承于Button，给它添加一个color的自定义属性。

xml 调用方式

```
 <leix.lebean.mvvm.widget.ExtButton
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:onClick="@{vModel.refreshClick}"
            android:text="Refresh"
            app:color="@{@color/colorPrimary}" />
```

ExtButton Java文件

```
**
 * Name:ExtButton
 * Description:
 * Author:leix
 * Time: 2017/4/7 09:48
 */
public class ExtButton extends Button {
    public ExtButton(Context context) {
        super(context);
    }
    public ExtButton(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    public ExtButton(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
    }

    public void setColor(@ColorInt int color) {
        setTextColor(color);
    }
}
```



## 给属性添加变化监听

​	当数据模型的某些属性值发生变化时，我们可能要做一些业务处理，那我们怎么才能捕捉到Model属性的变化呢？通过addOnPropertyChangedCallback可以捕捉到属性的改变。

```
author.addOnPropertyChangedCallback(new Observable.OnPropertyChangedCallback() {
            @Override
            public void onPropertyChanged(Observable observable, int i) {
                if (i == BR.name) {
    
                }
            }
        });
```



## 属性值变化时的动画

​	当模型属性值发生了变化，更新到UI的同时，我们希望有个动画，那我们可以给binding添加一个OnRebindCallback监听，然后在里里面对应方法添加动画效果。

```
binding.addOnRebindCallback(new OnRebindCallback() {
           @Override
           public boolean onPreBind(ViewDataBinding binding) {
               return super.onPreBind(binding);
           }

           @Override
           public void onCanceled(ViewDataBinding binding) {
               super.onCanceled(binding);
           }

           @Override
           public void onBound(ViewDataBinding binding) {
               super.onBound(binding);
           }
       });
```



## 一些隐式的表达

1. 重复表达

   ```
     <ImageView android:visibility="@{user.hasPhoto ? View.VISIBLE : View.GONE}"/>
           <TextView android:visibility="@{user.isAdult ? View.VISIBLE : View.GONE}"/>
           <CheckBox android:visibility="@{user.isAdult ? View.VISIBLE : View.GONE}"/>
   ```

   ```
   <ImageView android:id="@+id/firstImage" android:visibility="@{user.hasPhoto ? View.VISIBLE : View.GONE}"/>
   <TextView android:visibility="@{firstImage.visibility}"/>
   <CheckBox android:visibility="@{firstImage.visibility}"/>
   ```

2. 隐匿UI更新

   ```
   <CheckBox android:id="@+id/checkbox"/>
   <ImageView android:visibility="@{checkbox.checked ?View.VISIBLE : View.GONE}" />
   ```

   ​

## RecyclerView 的使用

   ​	这里介绍RecyclerView的使用，其它RecyclerView是使用适配器模式View的代表，讲了它的使用，ViewPager、Spiner这些控件的使用与它基本一致。
1. 创建item的layout文件

```
      <?xml version="1.0" encoding="utf-8"?>
      <layout xmlns:android="http://schemas.android.com/apk/res/android">
          <data class="RecyclerItemBinding">
              <variable
                  name="author"
                  type="cn.com.ssii.mvvm.book.model.Author" />
          </data>
          <LinearLayout
              android:layout_width="match_parent"
              android:layout_height="match_parent"
              android:orientation="vertical">
              <TextView
                  android:layout_width="match_parent"
                  android:layout_height="wrap_content"
                  android:text="@{author.name}" />
              
          </LinearLayout>
     </layout>
```

2. 写Adapter
```
      /**
       * Name:AuthorAdapter
       * Description:
       * Author:leix
       * Time: 2017/4/5 16:51
       */
      public class AuthorAdapter extends RecyclerView.Adapter<AuthorAdapter.ViewHolder> {
          List<Author> dataSet;

          @Override
          public ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
              ViewDataBinding binding = DataBindingUtil.inflate(LayoutInflater.from(parent.getContext()), R.layout.activity_author, null, false);
              return new ViewHolder(binding);
          }

          @Override
          public void onBindViewHolder(ViewHolder holder, int position) {
              holder.getBinding().setVariable(BR.author,dataSet.get(position));
          }

          @Override
          public int getItemCount() {
              return dataSet == null ? 0 : dataSet.size();
          }

          public static class ViewHolder<T extends ViewDataBinding> extends RecyclerView.ViewHolder {
              protected T binding;

              public ViewHolder(T binding) {
                  super(binding.getRoot());
                  this.binding = binding;
              }

              public T getBinding() {
                  return binding;
              }
          }
      }
```



## 一些有用的注解

### @BindingAdapter
​	BindingAdapter可以实现一些自定义属性设置的操作。

​	假如我们要在一个ImageView上显示一张网络图片，原生的方法只有个android:src，但是这个方法是没法加载显示一个网络图片的，但是我们也不想重写ImageView，这时BindAdapter方法就有神奇作用了。先定义一个图片处理的类，并定义一个图片加载的方法，要注意的时方法必需是静态方法，否侧会出错的。

```
@BindingAdapter({"image"})
public static void loadImage(ImageView view, String url) {

}
```

​	在这个方法中写自己的图片加载方法，根据选择的图片加载库不同而不同，ImageLoader,Glide等。

​	在xml文件中使用app:image属性设置图片地址，记得在layout节点中添加 xmlns:app="http://schemas.android.com/apk/res-auto"

```
<ImageView
	app:image="@{vModel.book.picture}"
	android:layout_width="match_parent"
	android:layout_height="200dp" />
```

​	这个方法的用处很大，前面讲的RecyclerView的数据加载只是单纯的在RecyclerView中根据数据动态更新UI显示，如果我在一个Activity里有RecyclerView和其它控件，整个界面的数据与服务器由一个http请求完成，那么怎么实现服务请求完成后更新整个界面数据呢？我的解决方案就是使用BindingAdapter，给RecyclerView设一个data属性，data属性绑定整个数据模型中的一个List字段值，然后在BindingAdater方法中更新Adapter数据源来更新RecyclerView的数据内容。

### @Bindable

​	被@Bindable注解的get方法对应的变量会在BR资源中生成一条记录，它和notifyPropertyChanged(BR.*)配合起来可实现数据模型值修改后自动更新到UI

```
/**
 * Name:Author
 * Description:
 * Author:leix
 * Time: 2017/4/5 14:57
 */
public class Author extends BaseObservable {
    private String name;
    private Date birthdate;
    private boolean male;

    @Bindable
    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
        notifyPropertyChanged(BR.name);
    }

    @Bindable
    public Date getBirthdate() {
        return birthdate;
    }
    public void setBirthdate(Date birthdate) {
        this.birthdate = birthdate;
        notifyPropertyChanged(BR.birthdate);
    }

    @Bindable
    public boolean isMale() {
        return male;
    }
    public void setMale(boolean male) {
        this.male = male;
        notifyPropertyChanged(BR.male);
    }
}
```

> 内容就写这么多吧，希望对大家有些帮助！