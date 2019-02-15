#### ViewPgaer使用

* 基本使用

  实现PagerAdapter，instantiateItem方法返回创建的View交给ViewPager显示

  > 参考 https://blog.csdn.net/suyimin2010/article/details/80659993

* 结合Fragment联动

  1. 实现FragmentPageAdapter或者FragmentStatePageAdapter
  2. 构建FragmentList

  > 参考 https://blog.csdn.net/harvic880925/article/details/38660861

#### ViewPager缓存机制

ViewPager可以通过setOffscreenPageLimit方法设置缓存页数，默认缓存页数==1，效果就是ViewPager会保证当前显示页面的左右两个页面，处于准备好的状态，一划过来就可以显示。

如果我们把这个值设置大一点，左右两边的页面就会保存多一点。

> tip1 : 页面的销毁，是指调用了Fragment的onDestroyView，此回调走了之后，再想显示Fragment，就会重走onCreateView，这个待会儿详细说

下面我们来看下ViewPager是怎么管理页面缓存的

```java
    public void setOffscreenPageLimit(int limit) {
        if (limit < 1) {
            Log.w("ViewPager", "Requested offscreen page limit " + limit + " too small; defaulting to " + 1);
            limit = 1;
        }
        if (limit != this.mOffscreenPageLimit) {
            this.mOffscreenPageLimit = limit;
            this.populate();
        }
    }
```

上面这段代码是设置缓存页数，设置了之后，调了populate，我们再继续看这个populate，时间有限，我们只看核心部分..

```java
void populate(int newCurrentItem) {
    //下面在计算要显示的起始位置和终止位置
    final int startPos = Math.max(0, mCurItem - pageLimit);
    final int N = mAdapter.getCount();
    final int endPos = Math.min(N - 1, mCurItem + pageLimit);
    /**
    *下面这段代码干了两件事情，算出哪些页面不需要显示之后，
    * 1. 从VP自己保存的list里面删掉它
    * 2. 让adapter销毁它，adapter如何销毁的，下面讲adapter再讲
    * 大概就是这么个流程，具体怎么算的，忽略
    */
    mItems.remove(itemIndex);
    mAdapter.destroyItem(this, pos, ii.object);
}

```

#### ViewPager动画

VP里面的页面，切换的时候，可以加上点动画，看起来更酷炫，面试官问到几率不大，大家了解一下即可

调setPageTransformer(boolean reverseDrawingOrder, PageTransformer transformer)方法即可，此方法第一个参数是是否要翻转动画，具体效果没试过，第二个参数，是一个接口，接口里面会传一个position，和页面view，可以根据position从-1到1的值，进行计算，实现你想要的动画，谷歌给了两个已经实现的

> DepthPageTransformer
>
> ZoomOutPageTransformer
>
> 效果参考 https://blog.csdn.net/weixin_39251617/article/details/79399592

#### ViewPager两种Adapter

前面提到Fragment列表可以用两个Adapter来管理，分别是FragmentPageAdapter和FragmentStatePageAdapter，这两个adapter的方法都差不多，差别如下 : 

前面我们再ViewPager缓存机制里面提到，ViewPager会计算不需要显示的Fragment，通过adapter调用destroyItem。

FragmentPageAdapter的destroyItem方法，只是将Fragment从UI中移除掉，但Fragment实例活在FragmentManager中，此时被移除掉的Fragment只会走到onDestroyView，下次显示的时候，会走一次onDestroyView。代码如下 : 

```java
    public void destroyItem(ViewGroup container, int position, Object object) {
        if (mCurTransaction == null) {
            mCurTransaction = mFragmentManager.beginTransaction();
        }
        if (DEBUG) Log.v(TAG, "Detaching item #" + getItemId(position) + ": f=" + object
                + " v=" + ((Fragment)object).getView());
        //看到没，只是detach
        mCurTransaction.detach((Fragment)object);
    }
```



FragmentStatePageAdapter不仅会从UI移除掉，并且会从FragmentManager中移除，会走到Fragment的onDetach。下次显示Fragment的时候，会重新创建Fragment。代码如下 : 

```java
    public void destroyItem(ViewGroup container, int position, Object object) {
        Fragment fragment = (Fragment) object;

        if (mCurTransaction == null) {
            mCurTransaction = mFragmentManager.beginTransaction();
        }
        if (DEBUG) Log.v(TAG, "Removing item #" + position + ": f=" + object
                + " v=" + ((Fragment)object).getView());
        while (mSavedState.size() <= position) {
            mSavedState.add(null);
        }
        mSavedState.set(position, fragment.isAdded()
                ? mFragmentManager.saveFragmentInstanceState(fragment) : null);
        mFragments.set(position, null);
		//看到没，下面是remove
        mCurTransaction.remove(fragment);
    }
```

#### Fragment基本使用及生命周期

* 基本使用

  Fragment设计之初，是为了手机和平板，在不同的显示方式上，可以板块的，碎片化的显示一些布局。

  可以在布局文件中写Fragment，也可以手动new一个Fragment，交给FragmentManager来显示，来段示例代码:

  ```java
  getSupportFragmentManager()
          .beginTransaction()
          .add(R.id.fragment_container, new MyFragment());
  ```

  上面就是最简单的，显示一个Fragment，其中R.id.fragment_container是activity布局里面的一个Fragment容器ViewGroup。

* 生命周期，给大家看一张图吧

  ![img](https://images2015.cnblogs.com/blog/945877/201611/945877-20161123093212096-2032834078.png)

  ​	光看图可能不清楚，我们结合ViewPager一起看下嘛，直接看日志，，

  ```
  TestFragment是第一个Fragment，下面这段，是TestFragment1刚显示出来的时候，会如期走到onResume，其实这个时候，TestFragment2也走到了onResume，只是没有设置可见
  -----------------------------------------------------------------
  2019-02-15 15:34:15.339 29569-29569/? I/TestFragment1: setUserVisibleHint: false
  2019-02-15 15:34:15.339 29569-29569/? I/TestFragment1: setUserVisibleHint: true
  2019-02-15 15:34:15.340 29569-29569/? I/TestFragment1: onAttach: 
  2019-02-15 15:34:15.340 29569-29569/? I/TestFragment1: onCreate: 
  2019-02-15 15:34:15.341 29569-29569/? I/TestFragment1: onCreateView: 
  2019-02-15 15:34:15.344 29569-29569/? I/TestFragment1: onStart: 
  2019-02-15 15:34:15.344 29569-29569/? I/TestFragment1: onResume: 
  
  
  下面这段，是滑动到TestFragment2的时候，TestFragment1只设置成了不可见，其他生命周期每走，因为显示TestFragment2的时候，要保证左右两边的Fragment时刻准备好的，划过来就可见
  ------------------------------------------------------------------
  2019-02-15 15:34:27.853 29569-29569/? I/TestFragment1: setUserVisibleHint: false
  
  下面这段，是滑动到TestFragment3的时候，TestFragment1不需要时刻准备着了，他可以休息了，就销毁了View，走到onDestroyView，省点内存，但因为我使用的是FragmentPagerAdapter，所以只走到onDestroyView,Fragment实例还在FragmentManager里面，不会销毁
  ------------------------------------------------------------------
  2019-02-15 15:34:31.951 29569-29569/? I/TestFragment1: onPause: 
  2019-02-15 15:34:31.951 29569-29569/? I/TestFragment1: onStop: 
  2019-02-15 15:34:31.951 29569-29569/? I/TestFragment1: onDestroyView: 
  
  下面这段，是我从TestFragment3滑到TestFragment2的时候，滑到2了，1就需要时刻准备好，所以创建好了view，走到resume。你看这里没有重新创建Fragment，就印证了上面的话
  ------------------------------------------------------------------
  2019-02-15 15:34:38.009 29569-29569/? I/TestFragment1: setUserVisibleHint: false
  2019-02-15 15:34:38.013 29569-29569/? I/TestFragment1: onCreateView: 
  2019-02-15 15:34:38.023 29569-29569/? I/TestFragment1: onStart: 
  2019-02-15 15:34:38.023 29569-29569/? I/TestFragment1: onResume: 
  
  下面这段，是从2滑到1的时候，仅需要把1设置可见就行了
  ------------------------------------------------------------------
  2019-02-15 15:34:40.461 29569-29569/? I/TestFragment1: setUserVisibleHint: true
  
  ```

  

#### Fragment懒加载

上面我们看了生命周期和Adapter，下面来说说市面上常见的懒加载是干嘛的，怎么实现的。

* 为什么要用懒加载？

  上面我看到的ViewPager+多个Fragment中，Fragment的生命周期有个奇怪的地方，就是Fragment还不可见的时候，就走了onResume了，然后滑过Fragment，销毁的时候，才会走onPause，如果我们要放请求数据，处理数据的代码，放在哪里合适呢？如果放在resume里面，就会造成，三个Fragment同时请求数据的情况，这样对性能不大好，并且，你什么时候去停止这些请求呢？页面都不可见了，你还在转圈？不大好吧，所以，这个时候懒加载就出来了，当页面真正可见的时候，才去请求，不可见，就停止请求，极大的节省资源。

* 如何实现

#### Fragment栈管理

#### Fragment常见问题



