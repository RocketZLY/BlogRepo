>做过Md风格app的朋友都应该用过Drawerlayout,而Drawerlayout的内容一般情况下是可以用support design包下的navigationview来实现。正如曾经的actionbar一样,看似非常完美了，，但当你希望去修改每个icon颜色或者每个文字颜色的时候却无法做到。正因如此本文主要介绍自定义navigationview的实现。还未使用过md风格控件的朋友可以先看看我另外一篇博客http://blog.csdn.net/zly921112/article/details/50733435

先来看我们要实现的效果
![这里写图片描述](http://img.blog.csdn.net/20160404154642876)
从navigationview源码发现,整个列表其实就是用recyclerview实现,那么同样我们也依葫芦画瓢,直接撸码

activity的布局文件
```
<?xml version="1.0" encoding="utf-8"?>
<android.support.v4.widget.DrawerLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/drawerlayout"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical">

    </LinearLayout>


    <android.support.v7.widget.RecyclerView
        android:id="@+id/rv_drawer"
        android:layout_width="@dimen/drawer_width"
        android:layout_height="match_parent"
        android:layout_gravity="start"
        android:background="@color/white" />

</android.support.v4.widget.DrawerLayout>

```
这里并没有太多东西就是将drawerlayout的左侧菜单设置为recyclerview

activity代码

```
package com.zly.www.bzmh.activity;

import android.os.Bundle;
import android.support.v4.view.GravityCompat;
import android.support.v4.widget.DrawerLayout;
import android.support.v7.widget.LinearLayoutManager;
import android.support.v7.widget.RecyclerView;

import com.zly.www.bzmh.R;
import com.zly.www.bzmh.adapter.DrawerAdapter;
import com.zly.www.bzmh.base.BaseActivity;

import butterknife.Bind;
import butterknife.ButterKnife;

public class MainActivity extends BaseActivity {

    @Bind(R.id.drawerlayout)
    DrawerLayout drawerlayout;
    @Bind(R.id.rv_drawer)
    RecyclerView rvDrawer;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        ButterKnife.bind(this);
        init();
    }

    private void init() {
        //抽屉初始化
        DrawerAdapter drawerAdapter = new DrawerAdapter();
        drawerAdapter.setOnItemClickListener(new MyOnItemClickListener());
        rvDrawer.setLayoutManager(new LinearLayoutManager(this));
        rvDrawer.setAdapter(drawerAdapter);
    }

    /**
     * drawer item 点击事件
     */
    public class MyOnItemClickListener implements DrawerAdapter.OnItemClickListener {

        @Override
        public void itemClick(DrawerAdapter.DrawerItemNormal drawerItemNormal) {
            switch (drawerItemNormal.titleRes) {
                case R.string.drawer_menu_home://首页
                    break;
                case R.string.drawer_menu_rank://排行榜
                    break;
                case R.string.drawer_menu_column://栏目
                    break;
                case R.string.drawer_menu_search://搜索
                    break;
                case R.string.drawer_menu_setting://设置
                    break;
                case R.string.drawer_menu_night://夜间模式
                    break;
                case R.string.drawer_menu_offline://离线
                    break;
            }
            drawerlayout.closeDrawer(GravityCompat.START);
        }
    }
}
```

activity同样代码也并不多,给左侧的recyclerview设置adapter实现adapter中item点击接口

接下来就是adapter代码了,前方高能,在这里将会有比较多的代码了

```
package com.zly.www.bzmh.adapter;

import android.support.v7.widget.RecyclerView;
import android.view.LayoutInflater;
import android.view.View;
import android.view.ViewGroup;
import android.widget.ImageView;
import android.widget.TextView;

import com.facebook.drawee.view.SimpleDraweeView;
import com.zly.www.bzmh.R;

import java.util.Arrays;
import java.util.List;

/**
 * 抽屉adapter
 * Created by zly on 2016/3/30.
 */
public class DrawerAdapter extends RecyclerView.Adapter<DrawerAdapter.DrawerViewHolder> {

    private static final int TYPE_DIVIDER = 0;
    private static final int TYPE_NORMAL = 1;
    private static final int TYPE_HEADER = 2;

    private List<DrawerItem> dataList = Arrays.asList(
            new DrawerItemHeader(),
            new DrawerItemNormal(R.mipmap.icon_drawerlayout_home, R.string.drawer_menu_home),
            new DrawerItemNormal(R.mipmap.icon_drawerlayout_rank, R.string.drawer_menu_rank),
            new DrawerItemNormal(R.mipmap.icon_drawerlayout_column, R.string.drawer_menu_column),
            new DrawerItemNormal(R.mipmap.icon_drawerlayout_search, R.string.drawer_menu_search),
            new DrawerItemNormal(R.mipmap.icon_drawerlayout_setting, R.string.drawer_menu_setting),
            new DrawerItemDivider(),
            new DrawerItemNormal(R.mipmap.icon_drawerlayout_night, R.string.drawer_menu_night),
            new DrawerItemNormal(R.mipmap.icon_drawerlayout_offline, R.string.drawer_menu_offline)
    );


    @Override
    public int getItemViewType(int position) {
        DrawerItem drawerItem = dataList.get(position);
        if (drawerItem instanceof DrawerItemDivider) {
            return TYPE_DIVIDER;
        } else if (drawerItem instanceof DrawerItemNormal) {
            return TYPE_NORMAL;
        }else if(drawerItem instanceof DrawerItemHeader){
            return TYPE_HEADER;
        }
        return super.getItemViewType(position);
    }

    @Override
    public int getItemCount() {
        return (dataList == null || dataList.size() == 0) ? 0 : dataList.size();
    }

    @Override
    public DrawerViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        DrawerViewHolder viewHolder = null;
        LayoutInflater inflater = LayoutInflater.from(parent.getContext());
        switch (viewType) {
            case TYPE_DIVIDER:
                viewHolder = new DividerViewHolder(inflater.inflate(R.layout.item_drawer_divider, parent, false));
                break;
            case TYPE_HEADER:
                viewHolder = new HeaderViewHolder(inflater.inflate(R.layout.item_drawer_header, parent, false));
                break;
            case TYPE_NORMAL:
                viewHolder = new NormalViewHolder(inflater.inflate(R.layout.item_drawer_normal, parent, false));
                break;
        }
        return viewHolder;
    }

    @Override
    public void onBindViewHolder(DrawerViewHolder holder, int position) {
        final DrawerItem item = dataList.get(position);
        if (holder instanceof NormalViewHolder) {
            NormalViewHolder normalViewHolder = (NormalViewHolder) holder;
            final DrawerItemNormal itemNormal = (DrawerItemNormal) item;
            normalViewHolder.iv.setBackgroundResource(itemNormal.iconRes);
            normalViewHolder.tv.setText(itemNormal.titleRes);

            normalViewHolder.view.setOnClickListener(new View.OnClickListener() {
                @Override
                public void onClick(View v) {
                    if(listener != null){
                        listener.itemClick(itemNormal);

                    }
                }
            });
        }else if(holder instanceof HeaderViewHolder){
            HeaderViewHolder headerViewHolder = (HeaderViewHolder) holder;
        }

    }

    public OnItemClickListener listener;

    public void setOnItemClickListener(OnItemClickListener listener){
        this.listener = listener;
    }

    public interface OnItemClickListener{
        void itemClick(DrawerItemNormal drawerItemNormal);
    }




    //-------------------------item数据模型------------------------------
    // drawerlayout item统一的数据模型
    public interface DrawerItem {
    }


    //有图片和文字的item
    public class DrawerItemNormal implements DrawerItem {
        public int iconRes;
        public int titleRes;

        public DrawerItemNormal(int iconRes, int titleRes) {
            this.iconRes = iconRes;
            this.titleRes = titleRes;
        }

    }

    //分割线item
    public class DrawerItemDivider implements DrawerItem {
        public DrawerItemDivider() {
        }
    }

    //头部item
    public class DrawerItemHeader implements DrawerItem{
        public DrawerItemHeader() {
        }
    }



    //----------------------------------ViewHolder数据模型---------------------------
    //抽屉ViewHolder模型
    public class DrawerViewHolder extends RecyclerView.ViewHolder {

        public DrawerViewHolder(View itemView) {
            super(itemView);
        }
    }

    //有图标有文字ViewHolder
    public class NormalViewHolder extends DrawerViewHolder {
        public View view;
        public TextView tv;
        public ImageView iv;

        public NormalViewHolder(View itemView) {
            super(itemView);
            view = itemView;
            tv = (TextView) itemView.findViewById(R.id.tv);
            iv = (ImageView) itemView.findViewById(R.id.iv);
        }
    }

    //分割线ViewHolder
    public class DividerViewHolder extends DrawerViewHolder {

        public DividerViewHolder(View itemView) {
            super(itemView);
        }
    }

    //头部ViewHolder
    public class HeaderViewHolder extends DrawerViewHolder {

        private SimpleDraweeView sdv_icon;
        private TextView tv_login;

        public HeaderViewHolder(View itemView) {
            super(itemView);
            sdv_icon = (SimpleDraweeView) itemView.findViewById(R.id.sdv_icon);
            tv_login = (TextView) itemView.findViewById(R.id.tv_login);
        }
    }
}

```

看着代码很多其实内容相对简单,这里也就是根据数据结构来加载对应的布局.并将条目点击事件暴露出来.这里为了方便我把数据直接写在了adapter内部,也可以在activity中直接从adapter构造方法传入.

分割线布局item_drawer_divider.xml
```
<?xml version="1.0" encoding="utf-8"?>
<View xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="1dp"
    android:background="?android:attr/listDivider"/>

```
头部布局item_drawer_header.xml

```
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:fresco="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="@dimen/drawer_header_height"
    android:background="?attr/colorPrimary"
    android:orientation="vertical">

    <com.facebook.drawee.view.SimpleDraweeView
        android:id="@+id/sdv_icon"
        android:layout_width="@dimen/drawer_header_icon_size"
        android:layout_height="@dimen/drawer_header_icon_size"
        android:layout_centerVertical="true"
        android:layout_marginLeft="10dp"
        android:layout_marginTop="10dp"
        fresco:placeholderImage="@mipmap/icon_bg"
        fresco:roundAsCircle="true"/>

    <TextView
        android:id="@+id/tv_login"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_below="@+id/sdv_icon"
        android:layout_marginLeft="10dp"
        android:layout_marginTop="10dp"
        android:text="登录"
        android:textColor="@color/white"
        android:textSize="@dimen/text_micro" />
</RelativeLayout>
```
列表布局item_drawer_normal.xml

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="@dimen/drawer_item_height"
    android:gravity="center_vertical"
    android:orientation="horizontal">

    <ImageView
        android:id="@+id/iv"
        android:layout_width="@dimen/drawer_item_icon_size"
        android:layout_height="@dimen/drawer_item_icon_size"
        android:layout_marginLeft="@dimen/drawer_item_icon_leftmargin" />

    <TextView
        android:id="@+id/tv"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginLeft="@dimen/drawer_item_text_leftmargin"
        android:textColor="@color/black"
        android:textSize="@dimen/text_small" />
</LinearLayout>

```
资源文件
strings.xml
```
<resources>
    <!-- drawer_menu_text-->
    <string name="drawer_menu_home">首页</string>
    <string name="drawer_menu_rank">排行榜</string>
    <string name="drawer_menu_column">栏目</string>
    <string name="drawer_menu_search">搜索</string>
    <string name="drawer_menu_setting">设置</string>
    <string name="drawer_menu_night">夜间模式</string>
    <string name="drawer_menu_offline">离线</string>
</resources>
```
dimens.xml

```
<resources>
    <!-- drawerlayoutsize-->
    <dimen name="drawer_width">260dp</dimen>
    <dimen name="drawer_header_height">140dp</dimen>
    <dimen name="drawer_header_icon_size">50dp</dimen>
    <dimen name="drawer_item_icon_size">14dp</dimen>
    <dimen name="drawer_item_height">44dp</dimen>
    <dimen name="drawer_item_icon_leftmargin">14dp</dimen>
    <dimen name="drawer_item_text_leftmargin">38dp</dimen>


    <!-- 字体大小 -->
    <dimen name="text_micro">12sp</dimen>
    <dimen name="text_small">14sp</dimen>
    <dimen name="text_medium">18sp</dimen>
    <dimen name="text_large">20sp</dimen>
</resources>

```
到此已经可以实现navigationview一样的效果了,但是到此还有一点小小的瑕疵就是条目点击的波纹效果

可以通过如下代码设置波纹的背景：
A:   android:background="?android:attr/selectableItemBackground"波纹有边界
B:   android:background="?android:attr/selectableItemBackgroundBorderless"波纹超出边界

效果A:
![这里写图片描述](http://img.blog.csdn.net/20160404162952877)

效果B:
![这里写图片描述](http://img.blog.csdn.net/20160404163003284)

显然只需要给列表布局加上android:background="?android:attr/selectableItemBackground"即可完成点击波纹效果
而这个波纹的颜色我们可以通过修改styles.xml中colorControlHighlight(android:colorControlHighlight：设置波纹颜色)属性来调节动画颜色，从而可以适应不同的主题,但有一点要注意这个波纹效果只能在5.0以上有效,5.0以下就跟普通选择器一样.
