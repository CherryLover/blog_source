---
title: 看一看 DataBinding
---

![](https://monster-image-backup.oss-cn-shanghai.aliyuncs.com/picgo/blog/paul-schafer-t6oZEgL0z18-unsplash.jpg)

# 看一看 DataBinding

最近在看一些人写的代码，发现很多都有用到 DataBinding，每每看到 DataBinding 相关的代码看不懂，摸不透，实在是难受啊，可是怎么办呢，谁让自己不会的呢。没办法开始学吧，不要求会太多，总得能看的懂人家写的是啥意思吧。

> 本篇难度属于 DataBinding 入门级别，如果有什么错误，还请指正。要学习 DataBinding 最好的教材还是通过官方给的[实例教程](https://codelabs.developers.google.com/codelabs/android-databinding/#0)。

正式开始之前呢，总是得添加依赖，不过，DataBinding 的依赖可不像 RxJava 或是其他库通过引入两行依赖，而是通过在 app.gradle 文件中引入下面的配置：

```groovy
android {
    ...
    dataBinding {
        enabled = true
    }
}
```

有了依赖之后，我们就可以开始了，现在想象一个场景，我们需要把一段文字显示在 TextView 上。之前，我们都是通过 findViewById，然后 setText，现如今来一起看看 DataBinding 是怎么做的吧。

首先，我们需要把布局文件的结构进行改造一下，改造成这样的结构。

```xml
<layout>
    <data>
    </data>
    <ViewGroup>
    </ViewGroup>
</layout>
```

> Android Studio 对 DataBinding 有自动替换的支持。

![image-20190718101219778](https://monster-image-backup.oss-cn-shanghai.aliyuncs.com/picgo/blog/image-20190718101219778.png)

替换之后的 ViewGroup 就是之前的根布局，data 节点内用于定义数据。

```xml
<data>
    <variable name="content" type="String" />
</data>
```

以此形式定义的变量 content 等价于 `String content`。拿到了 content 这个变量，紧接着就需要设置到 TextView 上。设置 Text，使用的属性是 `android:text=""` 至于填入的值，就是 content，不过语法上稍有不同，`android:text="@{content}"` 其中 `@` 以及`{}`是用于包裹其中的值，以此来区分开 `android:text="content"`。

> Databinding 使用这种语法来设置内容，事实上这种表达方法不仅仅用于设置个字符串，还支持表达式，更多内容请参[考官方文档](https://developer.android.com/topic/libraries/data-binding/expressions)。本文也会介绍一些。

事实上，我们更多时候会把 content 作为一个类的成员变量。只要将 variable 的 type 属性改成对应的类名，同时在 TextView 上改为 `android:text="@{detail.content}"` 即可。

```xml
<data>
    <variable name="detail" type="me.monster.blogtest.model.MomentDetail" />
</data>
```

在布局文件中定义好了变量，现在我们该回到 Java 文件中将布局文件使用 DataBinding 进行绑定上数据了。

```java
public View onCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container,
                             @Nullable Bundle savedInstanceState) {
        FragmentDetailBinding viewDataBinding = DataBindingUtil.inflate(inflater, R.layout.fragment_detail, container, false);
        View root = viewDataBinding.getRoot();
        mMomentDetail = new MomentDetail();
  			mMomentDetail.setContent("使用 DataBinding");
        viewDataBinding.setDetail(mMomentDetail);
        return root;
    }
```

> 1. 这段代码是使用在 Fragment 中的，如果在 Activity 中，则使用 `DataBinding.Util.setContentView(Activity activity, int layoutId)` 这个方法。
> 2. FragmentDetailBinding 这个类是根据布局文件生成的，类名的规则是布局文件的文件名+Binding；

DataBinding 这样使用，好像不太够日常使用。当 mMomentDetail 内部的 content 发生改变时，TextView 能自动更新吗？暂时不行，想让 TextView 自动更新 mMomentDetail 的 content 的内容，我们还需要另外的一些东西。

要想及时更新 mMomentDetail 的 content 内容，按照以往的做法，就是在更新 mMomentDetail 的 content 内容时，手动调用 `setText()` 这个方法。现在有了 DataBinding 就不用了。我们可以把 mMomentDetail 的 content 定义为一种可观察其变化的类型，然后通过 DataBinding 就能自动完成对 TextView 上内容的更新。

```java
public class MomentDetail {    
    private String content;

    public String getContent() {
        return content == null ? "" : content;
    }

    public void setContent(String content) {
        this.content = content;
    }
}
```

这个是 TextView 不能自动更新内容的 MomentDetail 类，要想让 TextView 自动更新，只需要把 content 的类型改变即可。改为 `ObservableField<String> content`。

```java
private ObservableField<String> content = new ObservableField<>();

public ObservableField<String> getContent() {
    return content;
}

public String getContentValue() {
    return content.get();
}

public void setContent(String content) {
    this.content.set(content);
}
```

与普通的 String 类型不同的是，想要改变 content 的值，需要调用 `ObservableField.set(T t)` 方法，获取其中的值则使用 `ObservableField.get()` 方法。

DataBinding 还包含了一些其他的类，都是一样的用法。

- ObservableBoolean
- ObservableByte
- ObservableChar
- ObservableShort
- ObservableInt
- ObservableLong
- ObservableFloat
- ObservableDouble
- ObservableParcelable

![](https://monster-image-backup.oss-cn-shanghai.aliyuncs.com/picgo/blog/blog_databinding_observable.gif)

> 我在写这篇博客的时候，使用 Handler 延迟几秒钟对 MomentDetail 进行修改，用于模拟点击事件。

当我们用于观察的数据较多的时候，还可以通过将自定义的数据类继承 BaseObservable ，产生继承关系之后，数据类中的成员变量均可以在发生改变时自动应用到 View 上。

```java
public class MomentDetail extends BaseObservable {
    private String moment;
    private int goodCount;

    @Bindable
    public String getMoment() {
        return moment == null ? "" : moment;
    }

    @Bindable
    public int getGoodCount() {
        return goodCount;
    }

    public void setContent(String msg) {
        this.moment = msg;
        notifyPropertyChanged(BR.moment);
    }

    public void updateGood(int count) {
        goodCount = count;
        notifyPropertyChanged(BR.goodCount);
    }
}
```

跟其他的数据类没有什么太大的差距，只是多了一个注解 `@Bindable` 以及在成员变量进行更改后进行更新 `notifyPropertyChanged()`。不过，Google 还是推荐使用 LiveData 配合使用。

```java
public class MomentViewModel extends ViewModel {

    private MutableLiveData<String> name = new MutableLiveData<>("Abs");
    private MutableLiveData<Integer> good = new MutableLiveData<>();

    public void setNameValue(String value) {
        name.setValue(value);
    }

    public MutableLiveData<String> getName() {
        return name;
    }

    public MutableLiveData<Integer> getGood() {
        return good;
    }

    public void setGoodValue(int goodValue) {
        good.setValue(goodValue);
    }
}
```

相应的，布局文件中也要进行更改数据类型。

```xml
<variable
    name="viewModel"
    type="me.monster.blogtest.model.MomentViewModel" />
```

DataBinding 设置 View 的 `text`，`textColor` 等属性都可以直接通过类似 `android:text="@{viewModel.name}"` 这种形式来进行操作，那如果想对控件进行点击事件的监听怎么办呢？也不麻烦，我们只需要提供一个用于处理点击结果的方法即可。

例如，我们有一个 TextView 用于进行点击，并将点击次数显示在当前 TextView 上。

```java
public void upGood() {
    good.setValue(good.getValue() == null ? 1 : good.getValue() + 1);
}
```

```xml
<TextView
    android:id="@+id/iv_good"
    android:layout_width="wrap_content"
    android:layout_height="40dp"
    android:drawableStart="@drawable/ic_good"
    android:drawablePadding="8dp"
    android:gravity="center"
    android:onClick="@{() -> viewModel.upGood()}"
    android:text="@{String.valueOf(viewModel.good)}" />
```

不过，DataBinding 虽然挺好的，但是还是有些不足，比如通过 Glide 这种第三方框架加载图片，还是得 `findViewById`，事实上，DataBinding 提供了另一种解决思路，通过将一个静态方法加上注解，然后就可以在 View 中进行使用。

```java
@BindingAdapter({"imageRes"})
public static void loadImage(ImageView imageView, int resId) {
    Glide.with(imageView)
            .load(resId)
            .into(imageView);
}
```

```xml
<ImageView
    android:id="@+id/iv_avatar"
    imageRes="@{viewModel.avatar}"
    android:layout_width="50dp"
    android:layout_height="50dp"
    android:src="@mipmap/avatar_1" />
```

> BindAdapter 还有更多其他用法，请[参考官方文档](https://developer.android.com/topic/libraries/data-binding/binding-adapters)。

整体来说，DataBinding 不难，而且入门也挺简单的，不过，刚上手，很多时候摸不到头脑。不过，它的好处也很明显，在 Fragment 或是 Activity 中没有使用任何直接或是间接的 `findViewById` 操作。保证了 Activity 与 Fragment 的简洁，使其能更好的专注于页面的跳转及数据的处理，倘若配合着 Jetpack 的其他组件协同开发，估计会更高效些。

本文旨在学习如何简单的使用 DataBinding，主体内容较为简单，实际上，日常开发中还会经常用到 RecyclerView 及其他功能，DataBinding 也能提供很好的支持，有兴趣或是需要的话可以自行查阅[官方文档](https://developer.android.com/topic/libraries/data-binding)。

文章首发于[个人博客](https://jiangjiwei.site)，文中所有代码均已上传至 [GitHub](https://github.com/CherryLover/BlogTest)，代码分支为：dataBinding。

相关文章推荐：
- [用 MotionLayout 来做过渡动画](https://juejin.im/post/5d2afcfef265da1b8a4f4d0e)
- [来学一波 Navigation](https://juejin.im/post/5d171ec4f265da1b8f1ad690)

本文封面图：Photo by [Paul Schafer](https://unsplash.com/@paul__schafer?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/search/photos/learn?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)。