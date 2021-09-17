---
title: Navigation 之 Fragment 切换
---

![封面图](https://monster-image-backup.oss-cn-shanghai.aliyuncs.com/picgo/blog/blog_nav_state.jpg)

# Navigation 之 Fragment 切换

本篇是有关 Navigation 的第二篇，如有对 Navigation 不了解的朋友请先阅读[来学一波 Navigation](https://jiangjiwei.site/post/lai-xue-yi-bo-navigation/)。

## 多次执行 onCreateView

在上一篇中，我们利用 Navigation 与 BottomNavigationView 做出了一个有三个 Tab 的页面，分别是 Feed、Timer、Mine，这三个 Fragment 都是只在当前页面显示各自的名称。

现在我们来给 TimerFragment 加点内容，我们在 TimerFragment 的 onCreateView 方法中启动一个倒计时。

```java
private void startTimer() {
    new CountDownTimer(10 * 1000, 1000) {

        @Override
        public void onTick(long millisUntilFinished) {
            tvLabel.setText(String.valueOf((millisUntilFinished / 1000) + 1));
        }

        @Override
        public void onFinish() {
            tvLabel.setText("Finished");
        }
    }.start();
}
```

<img src="https://monster-image-backup.oss-cn-shanghai.aliyuncs.com/picgo/blog/blog_nav_tab.gif" style="zoom:33%;" />

仔细看上面的效果可以看到，每次切换到 TimerFragment 时，倒计时总会重新开始，不是我们想要的仅开始一次。这是什么问题导致的呢？答案是 TimerFragment 执行了多次的 onCreateView，为什么是会执行多次，Fragment 为什么会加载多次？我们没有什么特殊的操作呀。是不是因为 Navigation？

现在让我们深入到 Navigation 的源码看一看这到底是怎么一回事，以及我们该如何解决这一问题。

首先，我们需要明确我们的方向，就是 Navigation 到底是怎么做 Fragment 切换的，为什么会导致 Fragment 的 onCreateView 被多次执行。

从哪里作为入口呢？了解过 Navigation 的朋友对下面这行代码应该不会陌生，就是通过一个 View 获取到 NavController，然后通过执行 NavController 的 navigate 这个方法，我们就从这个方法开始。

```java
Navigation.findNavController(view)
        .navigate(id);
```

这个 navigate 有多个重载方法，我们开始的 navigate 方法最终也是执行到下面这个重载方法。

```java
navigate(NavDestination node, Bundle args, NavOptions navOptions, Navigator.Extras navigatorExtras)
```

方法的具体内容如下图：

![](https://monster-image-backup.oss-cn-shanghai.aliyuncs.com/picgo/blog/blog_code_navigate.png)

其中在第 9 行，我们可以看到通过 mNavigatorProvider 获取到了一个泛型类型为 NavDestination 的 Navigator 对象，并且在第 12 行时，通过调用刚获取到的 navigator 的 navigate 方法，得到了 NavDestination 这个对象。

这两行是关键代码，一个是获取到执行 navigator 的对象，一个是实际执行 navigate 的方法。看到这，我们就只需要找到 Navigator 的 navigate 方法即可。不过，Navigator 这个只是一个抽象类，我们还需要继续寻找它的实现类。

> 快捷键：Implementation(s) Mac: option(⌥) + command(⌘)+B

![image-20190724110340568](https://monster-image-backup.oss-cn-shanghai.aliyuncs.com/picgo/blog/image-20190724110340568.png)

Navigator 抽象类的关键代码：

```java
public abstract class Navigator<D extends NavDestination> {
    @Retention(RUNTIME)
    @Target({TYPE})
    @SuppressWarnings("UnknownNullness")
    public @interface Name {
        String value();
    }

    @NonNull
    public abstract D createDestination();

    @Nullable
    public abstract NavDestination navigate(@NonNull D destination, @Nullable Bundle args,
            @Nullable NavOptions navOptions, @Nullable Extras navigatorExtras);

    public abstract boolean popBackStack();

    @Nullable
    public Bundle onSaveState() {
        return null;
    }

    public void onRestoreState(@NonNull Bundle savedState) {
    }

    public interface Extras {
    }
}
```
通过快捷键我们能找到多个实现类，有 ActivityNavigator、DialogFragmentNavigator 还有 FragmentNavigator 等，这里我们只关注 FragmentNavigator 这个类中的 navigate 这个方法。

![](https://monster-image-backup.oss-cn-shanghai.aliyuncs.com/picgo/blog/blog_code_navigate_real.png)

别看这么多代码，别害怕，其实关键部分的代码就是第 32 行 `ft.replace(mContainerId, frag)` 这里使用的是 FragmentTransaction 的 replace 方法，这个方法不用说了吧。 replace 是移除了相同 id 的 fragment 然后再进行 add 的。

所以，看到这，我们也就知道了，为什么 TimerFragment 的 onCreateView 方法会被执行多次了，原因就是在这。

## 规避 replace

找到原因了，那我们有什么方法去规避，或者说去绕过这个 replace 吗？答案是有的。

还记得刚才我们找的下面这行代码吧（忘记的，请看第一张代码图的第 9 行），刚才我说，通过 mNavigatorProvider 找到一个泛型类型为 NavDestination 的 Navigator 对象，那它实际上是怎么找到的呢？是通过 node.getNavigatorName() 然后找的，这个 node 是什么东西？以及 mNavigatorProvider.getNavigator 内部究竟发生了什么？

```java
Navigator<NavDestination> navigator = mNavigatorProvider.getNavigator(
        node.getNavigatorName());
```

实际上这里的 node 就是一个 NavDestination 对象，而一个 NavDestination 对象就是对应着 navigation graph 中的节点信息。我用来演示的 Demo 的 navigation graph 文件如下：

```xml
<?xml version="1.0" encoding="utf-8"?>
<navigation xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/tab_navigation"
    app:startDestination="@id/feedFragment">

    <fragment
        android:id="@+id/feedFragment"
        android:name="me.monster.blogtest.tab.FeedFragment"
        android:label="fragment_feed"
        tools:layout="@layout/fragment_feed" />
    <fragment
        android:id="@+id/timerFragment"
        android:name="me.monster.blogtest.tab.TimerFragment"
        android:label="fragment_timer"
        tools:layout="@layout/fragment_timer" />
    <fragment
        android:id="@+id/mineFragment"
        android:name="me.monster.blogtest.tab.MineFragment"
        android:label="fragment_mine"
        tools:layout="@layout/fragment_mine" />
</navigation>
```

node.getNavigatorName 返回的就是 fragment 节点的节点名称 `fragment`，而 getNavigator 其实内部就是维护了一个类型为 HashMap 的 mNavigators，这个 HashMap 存的 key 就是节点名称，value 就是抽象类 Navigator 的实现类。而与 fragment 对应的 FragmentNavigator 也存储在其中。

既然是存在一个 map，并从中取出相对于的 Navigator 实现类，那我们能不能创建一个类并实现 Navigator，然后将 key、value 添加到那个 HashMap 中。答案是可行的。在NavigatorProvider 这个类中有两个公共方法：

- addNavigator(Navigator navigator)
- addNavigator(String name, Navigator navigator)

其中，一个参数的 addNavigator 也是调用了 两个参数的 addNavigator 方法，那个 name 也就是 navigation graph 中 fragment 节点的节点名称，同时也是 Navigator 这个抽象类中注解 `Name` 定义的值。而且在 NavController 这个类（最初我们找到的 navigate 所在的类）中有一个 getNavigatorProvider() 方法。

![](https://monster-image-backup.oss-cn-shanghai.aliyuncs.com/picgo/blog/uml_blog_navigation.jpg)

看到这，关系应该就比较清楚了。所以，我们需要自己创建一个类，实现 Navigator 并为 Name 注解添加一个值，然后在使用 Navigation 这个模块的 Activity 获取到 NavController 并调用其 getNavigatorProvider 方法后再调用 addNavigator 即可。

## 自定义 Navigator

Github 上已经有一个演示自定义实现 Navigator 的项目了。这个项目是以 Kotlin 语言编写的。

项目地址： https://github.com/STAR-ZERO/navigation-keep-fragment-sample。

> 说起来这个项目还是 [Drakeet](https://github.com/drakeet) 在他的知识星球中分享的。感谢 Drakeet 的分享。

我根据按照他的代码写了一份 Java 版本的，并且在其中改了两行代码（注释部分）。注释的内容其实就是使用 FragmentTranslation 对 Fragment 进行控制。原作者写的是 detach 与 attach 方法，我改成了使用 hide 和 show 方法。

```java
@Navigator.Name("keep_state_fragment")
public class KeepStateNavigator extends FragmentNavigator {
    private Context context;
    private FragmentManager manager;
    private int containerId;

    public KeepStateNavigator(@NonNull Context context, @NonNull FragmentManager manager, int containerId) {
        super(context, manager, containerId);
        this.context = context;
        this.manager = manager;
        this.containerId = containerId;
    }

    @Nullable
    @Override
    public NavDestination navigate(@NonNull Destination destination, @Nullable Bundle args, @Nullable NavOptions navOptions, @Nullable Navigator.Extras navigatorExtras) {
        String tag = String.valueOf(destination.getId());
        FragmentTransaction transaction = manager.beginTransaction();
        boolean initialNavigate = false;
        Fragment currentFragment = manager.getPrimaryNavigationFragment();
        if (currentFragment != null) {
//            transaction.detach(currentFragment);
            transaction.hide(currentFragment);
        } else {
            initialNavigate = true;
        }
        Fragment fragment = manager.findFragmentByTag(tag);
        if (fragment == null) {
            String className = destination.getClassName();
            fragment = manager.getFragmentFactory().instantiate(context.getClassLoader(), className);
            transaction.add(containerId, fragment, tag);
        } else {
//            transaction.attach(fragment);
            transaction.show(fragment);
        }

        transaction.setPrimaryNavigationFragment(fragment);
        transaction.setReorderingAllowed(true);
        transaction.commitNow();
        return initialNavigate ? destination : null;
    }
}
```

> 这里的代码没有传递 Bundle 类型的 args 同时也破坏了在 navigation graph 中切换动画的设置，如需要，自行加上即可。可参考 FragmentNavigator 类中实现。
>
> 感谢 [Qwer](https://github.com/1196455690) 提出问题

注意，使用自定义 Navigator 的时候 navigation graph 需要把 fragment 节点名称改为 keep_state_fragment，并且在承载的 Activity 中进行设置并且还需要把 Activity 布局文件中 fragment 的 navGraph 属性移除。

```java
NavController navController = Navigation.findNavController(this, R.id.fragment3);
NavHostFragment navHostFragment = (NavHostFragment) getSupportFragmentManager().findFragmentById(R.id.fragment3);
KeepStateNavigator navigator = new KeepStateNavigator(this, navHostFragment.getChildFragmentManager(), R.id.fragment3);
navController.getNavigatorProvider().addNavigator(navigator);
navController.setGraph(R.navigation.tab_navigation);
```

最后来看一下使用自定义 Navigator 时的 TabActivity。

<img src="https://monster-image-backup.oss-cn-shanghai.aliyuncs.com/picgo/blog/blog_nav_tab_state.gif" style="zoom: 33%;" />

这样好像看起来结束了？其实并没有，我们只是刚刚开始。

首先，我先更正一下，在第一篇关于 Navigation 的博客中从 SettingFragment 返回到 RootFragment 那一段代码有些问题。

> 没看过那篇文章的不要着急，其实就是 A 调到 B，然后在 B 中触发一个点击事件，再从 B 返回到 A。返回的代码如下。

原代码为：

```java
btnToRoot.setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        Navigation.findNavController(btnToRoot)
                .popBackStack();
    }
});
```

这里在点击事件中，最后执行的是 `popBackStack`，其实不应该调用这个方法应该用 `navigateUp` 这个方法。

在第一篇博客中，Navigation Graph 中所有的节点名称都是 Fragment，如果我用上面这种 keep_state_fragment 的方式，会发生什么呢？

<img src="https://i.loli.net/2019/10/21/BAC6leVN1WhdaKJ.gif" style="zoom: 50%;" />

可以看到，在把 Navigation Graph 节点名替换为 keep_state_fragment 后，在 SettingFragment 点击返回并没有进行返回。这是为什么呢？我没干啥呀，怎么不好使了。

不行，我要看看 Navigation 源码里面到底怎么做的。于是我开始了 debug 之旅。后来，我发现在 `Navigation.findNavController(btnToRoot).navigateUp();` 内部判断了当前的返回栈个数是否为 1，结果让我很震惊，返回的竟然真的是 1。所以，navigateUp 就理所当然的返回 false，也就没能从 SettingFragment 回到 RootFragment 了。

> 下面的两段代码分别是：NavController#navigateUp 和 NavController#getDestinationCountOnBackStack

![](https://i.loli.net/2019/10/21/bu9EAoP5HNJ7y8v.png)

```java
private int getDestinationCountOnBackStack() {
    int count = 0;
    for (NavBackStackEntry entry : mBackStack) {
        if (!(entry.getDestination() instanceof NavGraph)) {
            count++;
        }
    }
    return count;
}
```

我查了一下 mBackStack 这个数据类型，发现它是一个栈，紧接着找到 mBackStack 入栈的方法。

![](https://i.loli.net/2019/10/17/s91Txif3vPM2JOj.png)

mBackStack#add 相关的方法一共有 4 个，第一个方法是在 NavController#NavController 方法中进行调用的，其余 3 个 add 相关方法均是在 NavController#navigate 内调用，而调用 add 方法之外有一个判空。判空的对象就是来自 Navigator#navigate 这个方法。

> NavController 的 navigate 方法，有删减。

```java
private void navigate(@NonNull NavDestination node, @Nullable Bundle args,
        @Nullable NavOptions navOptions, @Nullable Navigator.Extras navigatorExtras) {
    //......
    Navigator<NavDestination> navigator = mNavigatorProvider
           .getNavigator(node.getNavigatorName());
    Bundle finalArgs = node.addInDefaultArgs(args);
    NavDestination newDest = navigator.navigate(node, finalArgs,
            navOptions, navigatorExtras);
    if (newDest != null) {
        // The mGraph should always be on the back stack after you navigate()
        if (mBackStack.isEmpty()) {
            mBackStack.add(new NavBackStackEntry(mGraph, finalArgs));
        }
        // Now ensure all intermediate NavGraphs are put on the back stack
        // to ensure that global actions work.
        ArrayDeque<NavBackStackEntry> hierarchy = new ArrayDeque<>();
        NavDestination destination = newDest;
        while (destination != null && findDestination(destination.getId()) == null) {
            NavGraph parent = destination.getParent();
            if (parent != null) {
                hierarchy.addFirst(new NavBackStackEntry(parent, finalArgs));
            }
            destination = parent;
        }
        mBackStack.addAll(hierarchy);
        // And finally, add the new destination with its default args
        NavBackStackEntry newBackStackEntry = new NavBackStackEntry(newDest,
                newDest.addInDefaultArgs(finalArgs));
        mBackStack.add(newBackStackEntry);
    }
  //......
}
```

根据我们之前的经验，可以得出这里的 Navigator 就是我们自定义的 KeepStateNavigator 这个对象，那 navgate 这个方法的返回值也就是我们自己控制的，也就是我们自己给自己挖了个坑。2333~

来吧，来看一下刚才写的代码。

```java
public NavDestination navigate(Destination destination, Bundle args, NavOptions navOptions, Navigator.Extras navigatorExtras) {
    String tag = String.valueOf(destination.getId());
    FragmentTransaction transaction = manager.beginTransaction();
    boolean initialNavigate = false;
    Fragment currentFragment = manager.getPrimaryNavigationFragment();
    if (currentFragment != null) {
      transaction.hide(currentFragment);
    } else {
      initialNavigate = true;
    }
    Fragment fragment = manager.findFragmentByTag(tag);
    if (fragment == null) {
      String className = destination.getClassName();
      fragment = manager.getFragmentFactory().instantiate(context.getClassLoader(), className);
      transaction.add(containerId, fragment, tag);
    } else {
      transaction.show(fragment);
    }

    transaction.setPrimaryNavigationFragment(fragment);
    transaction.setReorderingAllowed(true);
    transaction.commitNow();
    return initialNavigate ? destination : null;
  }
```

在最后一行中，我们通过对 `initialNavigate` 进行判断然后返回 null 或是 destination 对象。而把 initialNavigate 赋值为 true 则是只有在 currentFragment 为空时才会进行，什么时候 currentFragment 才会为空？只有当打开一个 Activity 并为其填充第一个 Fragment 时才会为 true，在我们当前这个场景里，就是当应用启动，打开 RootFragment 时 initialNavigate 为 true，从 RootFragment 跳转到 SettingFragment 时 initialNavigate 为 false。

这显然是有问题的，那么我们需要改为，当这个 fragmen 为空时，在 `transaction.add(containerId, fragment, tag);` 之后把 initialNavigate 赋值为 true。这样一来，NavController#getDestinationCountOnBackStack 就能获取到实际的 fragment 大小了，也就不会直接 return fase 了。

运行一下看看结果？别着急啊，再检查检查。刚才我说在 `Navigation.findNavController(btnToRoot).navigateUp();` 内部判断了当前的返回栈个数是否为 1，现在我们把为 1 的情况解决了，那么当返回栈的个数不为 1 时它怎么做的？在判断返回栈个数不是 1 的后经过内部调用，最终来到了 `NavController#popBackStackInternal` 这个方法内。

> NavController 的 popBackStackInternal 方法，有删减

```java
boolean popBackStackInternal(@IdRes int destinationId, boolean inclusive) {
    if (mBackStack.isEmpty()) {
        // Nothing to pop if the back stack is empty
        return false;
    }
    ArrayList<Navigator> popOperations = new ArrayList<>();
    Iterator<NavBackStackEntry> iterator = mBackStack.descendingIterator();
    boolean foundDestination = false;
    while (iterator.hasNext()) {
        NavDestination destination = iterator.next().getDestination();
        Navigator navigator = mNavigatorProvider.getNavigator(
                destination.getNavigatorName());
        if (inclusive || destination.getId() != destinationId) {
            popOperations.add(navigator);
        }
    //......
    boolean popped = false;
    for (Navigator navigator : popOperations) {
        if (navigator.popBackStack()) {
            NavBackStackEntry entry = mBackStack.removeLast();
            popped = true;
     // ......
    return popped;
}
```

在这个方法内，我又看到了那个熟悉的面孔 navigator，在这里 navigator 执行了一个叫 popBackStack 的方法，这个方法看起来好像就是做返回事件的。可是，我们的 KeepStateNavigator 并没有这个方法啊，那是因为我们选择了继承自 FragmentNavigator，在 FragmentNavigator 有一套 popBackStack 逻辑，不过我们用不了。所以我们需要在 FragmentNavigator 进行重写这个方法。

由于我们需要返回到上一个页面，所以我们也得有个管理栈，然后在 KeepStateNavigator#navigate 方法中的 `transaction.add(containerId, fragment, tag);` 之后把当前 Fragment 添加到返回栈中，在 popBackStack 中根据一些条件再进行 remove 即可。

![](https://i.loli.net/2019/10/21/GaH4yIbd7uC6Bmz.png)

这样一来就可以了。

<img src="https://i.loli.net/2019/10/17/qn84IfTF7ybrViS.gif" style="zoom:50%;" />

> 不知道为什么，录制的 gif 画面一直在闪……

### 随意切换

好了，这样就可以了，终于可以愉快的使用 Navigation 了。直到有一天，老大找到我，跟我说了一个需求。

> 从 A 页面进入 B 页面，再从 B 页面进入 C 页面，在 C 页面产生一个事件，然后用户返回时，需要跳过 B，也就是从 C 直接回到 A。

问我这个能不能在 Navigation 上实现，我想了一下，说可以。下面就分享一下实现这种效果的思路，我个人觉得可以有两个解决方法，下面我依次来说一下。

> 假设页面打开顺序为：A、B、C。

- 第一种：优先关闭

  当从 C 返回到 A 时，其实并不一定是返回的时候进行操作，可能是在某个事件产生之后，这时就把 B 给关闭，此时回退栈里面也就只剩下 A 和 B。这时候只需要正常走页面返回逻辑即可。

- 第二种：优先返回

  当从 C 返回到 A 时，也可以直接跳过 B，具体方法为：当从 C 点击返回时，触发返回栈的操作，当完成返回操作后发现当前页面需要跳过时，则继续返回，此时也就回到了 A。

落实到 Navigation 中，就是在自定义 Navigator 中添加一些方法，然后在需要执行此类操作的地方获取到 Navigator 对象，并进行相关操作。

获取 Navigator 的相关代码如下：

```java
NavController navController = Navigation.findNavController(btnToRoot);
NavigatorProvider navigatorProvider = navController.getNavigatorProvider();
Navigator<?> navigator = navigatorProvider.getNavigator("keep_state_fragment");
if (navigator instanceof KeepStateNavigator) {
    ((KeepStateNavigator) navigator).closeMiddle(R.id.settingsFragment);
}
```

## 思考

我第一次学习 Navigation 的时候就瞄了一眼，觉得这不就是个 Fragment 的管理框架吗？有什么的呀，比其他Fragment 的管理框架好吗？看起来一般般啊。哇哦，好复杂啊，算了，不看了，知道大致怎么用就行。看着大家讨论的越来越多，没忍住，又去仔细看了一下 Navigation 的使用，以及稍微阅读了源码。不就是个 Fragment 管理框架吗？怎么搞这么复杂，什么 NavigatorProvider、Navigator、Destination 这些都是啥啊。

随着我看的内容越来越多，实践的也变多了，原生的 Navigation 越来不不能满足需求了，才发现，原来 Google 早就想到了，只是没有给我们提供具体的解决方法，只是把一些东西开放出来供开发者在不同场景下进行自定义使用。

想想自己项目里的代码，好像如果要扩展的话，就得改动较多原来的代码，不能像 Navigation 这样，在需要改动的时候，尽量不触动原有代码，而通过接口、Provider、泛型等更多是编码技巧或是设计模式上的技巧来完成业务需求。

也不应该把过多的心思花在各式各样的第三方库上，而是把更多的精力花在基础技能上，虽然可能一时半会儿看不出什么结果，但这可能是笑到最后的方法。





本文首发于[个人博客](https://jiangjiwei.site/)，文中全部源代码已上传至 [GitHub](https://github.com/CherryLover/BlogTest)，代码分支为 closeBefore。喜欢本文的麻烦点个🌟。

本文封面图：Photo by [João Silas](https://unsplash.com/@joaosilas?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/search/photos/translation?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)
