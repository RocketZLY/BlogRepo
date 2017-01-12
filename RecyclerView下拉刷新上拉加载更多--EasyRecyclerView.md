# EasyRecyclerView
## **描述**
这是一个下拉刷新上拉加载更多框架(ps:后期还会加入一些常用的功能.),头部用的秋哥的[android-Ultra-Pull-To-Refresh](https://github.com/liaohuqiu/android-Ultra-Pull-To-Refresh),底部和没有数据的状态自己实现的.其实刚刚开始我是想找个库直接用的,试了几个排名靠前的,感觉跟自己想要的不太一样,索性自己写了一个,当然这当中也遇到了问题,多亏[仲锦大师](https://github.com/chenzj-king)的帮助在此感谢.

感谢完了这里附上库的地址[EasyRecyclerView](https://github.com/zhuliyuan921112/EasyRecyclerView)

### 特点:
- 可定制的头部 (可以查看[android-Ultra-Pull-To-Refresh](https://github.com/liaohuqiu/android-Ultra-Pull-To-Refresh)文档)
- 可定制的底部 (加载中/没有数据/加载失败 三种状态的定制)
- 可定制的没有数据状态显示 (目前只有一个状态)
- 目前提供一个实现好的ItemDecoration(头部吸附效果)
</br>

## **效果预览**
### 1.定制头部&定制脚步
- 头部秋哥已经定制了很多样式就直接使用了
- 脚部这边使用的已经实现好的ErvDefaultFooter

默认头部与顶部效果

![](http://of1ktyksz.bkt.clouddn.com/erv_custom_all.gif)

material style头部

![](http://of1ktyksz.bkt.clouddn.com/erv_material_header.gif)
</br>

### 2.头部吸附
![](http://of1ktyksz.bkt.clouddn.com/erv_custom_decoration.gif)
</br>

## **使用方式**
### 依赖
gradle依赖(一切不能gradle依赖的库都是耍流氓^_^,所以我加上了)

	compile 'com.yysauce:easyrecyclerview:1.0.0' 

 library依赖


### 配置
目前有两个参数可以配置

- app:emply_layout</br>
  没有数据时候布局

- app:number_load_more</br>
  最后可见条目 + number_load_more > total 触发加载更多;默认值为4

### xml中配置示例  
    <com.zly.www.easyrecyclerview.EasyDefRecyclerView
            android:id="@+id/erv"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            app:emply_layout="@layout/erv_default_empty" />





### activity代码配置

    erv.setAdapter(rvAdapter = new RvAdapter());//设置adapter
    erv.setLastUpdateTimeRelateObject(this);//传入参数类名作为记录刷新时间key
    erv.setOnRefreshListener(this);//设置刷新监听
    erv.setOnLoadListener(this);//设置加载更多监听

由于这里使用的EasyDefRecyclerView,头部就是默认经典样式所以需要调用,使用其他头部时不需要调用

    erv.setLastUpdateTimeRelateObject(this);//传入参数类名作为记录刷新时间key


### adapter代码配置

adapter需要实现CommonAdapter或者MultipleAdapter抽象方法

    //创建ViewHolder
    public abstract VH createCustomViewHolder(ViewGroup parent, int viewType);
    //ViewHolder设置数据
    public abstract void bindCustomViewHolder(VH holder, T t, int position);

MultipleAdapter多条目布局还多一个方法需要实现

    //返回多条目的type
    public abstract int customItemViewType(int position);

目前提供了下面这些方法操作adapter数据,具体实现可以在CommonAdapter中查看

新增数据

- public void add(@NonNull T object)
- public void addAll(@NonNull Collection collection)
- public void addAll(@NonNull T... items)
- public void insert(@NonNull T object, int index)
- public void insertAll(@NonNull Collection collection, int index)

删除数据

- public void remove(int index)
- public boolean remove(@NonNull T object)
- public void clear()

修改数据

- public void update(@NonNull List mDatas)

查看数据

- public T getItem(int position)
- public int getPosition(T item)
- public List getData()

排序

- public void sort(Comparator comparator)

加载布局

- public View inflateView(@LayoutRes int resId, ViewGroup parent)

adapter中ViewHolder需要继承BaseViewHolder
</br>


## **其他配置**
### 头部吸附效果

      mItemDecoration = new StickItemDecoration(context,dataList) {
                  @Override
                  public String getTag(int position) {
                      return "吸附头部显示的文字";
                  }
        }
      erv.addItemDecoration(mItemDecoration);

这里StickItemDecoration提供了如下方法来定制吸附效果

    //设置吸附条目高度
    public void setStickHeight(int mStickHeight)
    //设置吸附条目背景
    public void setStickBackgroundColor(int mStickBackgroundColor)
    //设置吸附文字颜色
    public void setStickTextColor(int mStickTextColor)
    //设置吸附文字大小
    public void setStickTextSize(int mStickTextSize)
    //设置吸附文字leftmargin
    public void setStickTextoffset(int mStickTextoffset)

## **自定义**

头部使用秋哥的[android-Ultra-Pull-To-Refresh](https://github.com/liaohuqiu/android-Ultra-Pull-To-Refresh)
秋哥默认已经实现了3个头部

- MaterialHeader
- PtrClassicDefaultHeader
- StoreHouseHeader

一般情况下这些样式应该够了,如果有特殊需求可以自定义头部.然后erv.setHeaderView(view);

底部的话目前我只实现了一个ErvDefaultFooter,自定义的话需要实现ErvLoadUIHandle接口.写法可以参考ErvDefaultFooter

     public interface ErvLoadUIHandle {

        /**
         * 允许加载更多
         */
        int LOAD = 1;

        /**
         * 暂无更多数据
         */
        int NOMORE = 2;

        /**
         * 加载失败
         */
        int LOADFAIL = 3;

        /**
         * @return 获取底部当前状态
         */
        int getState();

        void onLoading();//loading状态实现

        void onNoMore();//没有数据状态实现

        void onLoadFail(OnLoadListener listener);//加载失败实现


    }

实现后调用setFooterView()方法设置
</br>

## **总结**
目前还在EasyRecyclerView还在优化欢迎各位提出你们宝贵的意见,例子可以参考Sample
</br>

## **感谢**
秋哥的[android-Ultra-Pull-To-Refresh](https://github.com/liaohuqiu/android-Ultra-Pull-To-Refresh)</br>
[仲锦大师](https://github.com/chenzj-king)的帮助
</br>

## **联系方式**
qq:1835556188</br>

## **License**
    Copyright (c) 2016 zhuliyuan

    Licensed under the Apache License, Version 2.0 (the "License");
    you may not use this file except in compliance with the License.
    You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS,
    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    See the License for the specific language governing permissions and
    limitations under the License.
    Come on, don't tell me you read that.
