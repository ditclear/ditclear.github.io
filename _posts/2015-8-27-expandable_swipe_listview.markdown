---
layout: post
title:  "Android时光轴：ExpandableListview结合SwipeLayout" 
date:   2015-8-27
---

<p class="intro"><span class="dropcap">最</span>近开发一款app的时候需要用到时光轴，就去了解了一下怎么去做，然后知道需要用 ExpandableListview .而ExpandaListview其实就相当于我们非常熟悉的listview,当然没listview那么多开源代码。</p>



###效果如下：

<div  align="center">  
 <img style="box-shadow:5px 0 5px;" src="{{ site.url }}/assets/img/expandableListview.jpg" alt="expandableListview">
</div>

**[app下载](http://dp.wdjcdn.com/files/phoenix/4.52.1.8061/wandoujia-wandoujia_organic_binded.apk?remove=2&append=%93%00eyJhcHBEb3dubG9hZCI6eyJkb3dubG9hZFR5cGUiOiJkb3dubG9hZF9ieV9wYWNrYWdlX25hbWUiLCJwYWNrYWdlTmFtZSI6InZpZW5hbi5hcHAuY2FyZGdhbGxlcnkifX0Wdj01B0000837625)**

一个简单的ExpandableListView和listview差不多，主要是adpter麻烦些：

```java
	public class StatusExpandAdapter extends BaseExpandableListAdapter {
	private LayoutInflater inflater = null;
	private List<GroupStatusEntity> groupList;

	/**
	 * 构造方法
	 * 
	 * @param context
	 * @param oneList
	 */
	public StatusExpandAdapter(Context context,
			List<GroupStatusEntity> group_list) {
		this.groupList = group_list;
		inflater = (LayoutInflater) context
				.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
	}

	/**
	 * 返回一级Item总数
	 */
	@Override
	public int getGroupCount() {
		// TODO Auto-generated method stub
		return groupList.size();
	}

	/**
	 * 返回二级Item总数
	 */
	@Override
	public int getChildrenCount(int groupPosition) {
		if (groupList.get(groupPosition).getChildList() == null) {
			return 0;
		} else {
			return groupList.get(groupPosition).getChildList().size();
		}
	}

	/**
	 * 获取一级Item内容
	 */
	@Override
	public Object getGroup(int groupPosition) {
		// TODO Auto-generated method stub
		return groupList.get(groupPosition);
	}

	/**
	 * 获取二级Item内容
	 */
	@Override
	public Object getChild(int groupPosition, int childPosition) {
		return groupList.get(groupPosition).getChildList().get(childPosition);
	}

	@Override
	public long getGroupId(int groupPosition) {
		// TODO Auto-generated method stub
		return groupPosition;
	}

	@Override
	public long getChildId(int groupPosition, int childPosition) {
		// TODO Auto-generated method stub
		return childPosition;
	}

	@Override
	public boolean hasStableIds() {
		// TODO Auto-generated method stub
		return false;
	}

	@Override
	public View getGroupView(int groupPosition, boolean isExpanded,
			View convertView, ViewGroup parent) {

		GroupViewHolder holder = new GroupViewHolder();

		if (convertView == null) {
			convertView = inflater.inflate(R.layout.group_status_item, null);
		}
		holder.groupName = (TextView) convertView
				.findViewById(R.id.one_status_name);

		holder.groupName.setText(groupList.get(groupPosition).getGroupName());

		return convertView;
	}

	@Override
	public View getChildView(int groupPosition, int childPosition,
			boolean isLastChild, View convertView, ViewGroup parent) {
		ChildViewHolder viewHolder = null;
		ChildStatusEntity entity = (ChildStatusEntity) getChild(groupPosition,
				childPosition);
		if (convertView != null) {
			viewHolder = (ChildViewHolder) convertView.getTag();
		} else {
			viewHolder = new ChildViewHolder();
			convertView = inflater.inflate(R.layout.child_status_item, null);
			viewHolder.twoStatusTime = (TextView) convertView
					.findViewById(R.id.two_complete_time);
		}
		viewHolder.twoStatusTime.setText(entity.getCompleteTime());

		convertView.setTag(viewHolder);
		return convertView;
	}

	@Override
	public boolean isChildSelectable(int groupPosition, int childPosition) {
		// TODO Auto-generated method stub
		return false;
	}

	private class GroupViewHolder {
		TextView groupName;
	}

	private class ChildViewHolder {
		public TextView twoStatusTime;
	}

}

```
然后在activity中初始化view就可以了

```java

	/**
	 * 初始化可拓展列表
	 */
	private void initExpandListView() {
		statusAdapter = new StatusExpandAdapter(context, getListData());
		expandlistView.setAdapter(statusAdapter);
		expandlistView.setGroupIndicator(null); // 去掉默认带的箭头
		expandlistView.setSelection(0);// 设置默认选中项

		// 遍历所有group,将所有项设置成默认展开
		int groupCount = expandlistView.getCount();
		for (int i = 0; i < groupCount; i++) {
			expandlistView.expandGroup(i);
		}

		expandlistView.setOnGroupClickListener(new OnGroupClickListener() {

			@Override
			public boolean onGroupClick(ExpandableListView parent, View v,
					int groupPosition, long id) {
				// TODO Auto-generated method stub
				return true;
			}
		});
	}
```
以上不清楚的地方可以查看[这里](https://github.com/ljtyzhr/TimeLine "这里")

效果达到之后我在想是不是可以加入左右滑动的效果，因为很多listview都可以滑动编辑，可惜expandableListview没有那么多开源库，好在有[AndroidSwipeLayout](https://github.com/daimajia/AndroidSwipeLayout "这里")，这个库实现了任意方向的滑动效果，而且效果非常棒。

```java

	···
	@Override
	public View getChildView(final int groupPosition, final int childPosition,
			boolean isLastChild, View convertView, ViewGroup parent) {
		ChildViewHolder viewHolder = null;
		ChildStatusEntity entity = (ChildStatusEntity) getChild(groupPosition,
				childPosition);
		final CardModel model=new Select().from(CardModel.class)
										.where("Id=?",entity.getCardModel().getId()).executeSingle();
		Log.d("model", "" + model.getId() + model.toString());
		if (convertView != null) {
			viewHolder = (ChildViewHolder) convertView.getTag();
		} else {
			viewHolder = new ChildViewHolder();
			convertView = inflater.inflate(R.layout.child_status_item, null);
			viewHolder.card_type= (ImageView) convertView.findViewById(R.id.iv_type);
			viewHolder.twoStatusTime = (TextView) convertView
					.findViewById(R.id.two_complete_time);
			viewHolder.swipeLayout = (SwipeLayout) convertView.findViewById(R.id.sample);
			viewHolder.swipeLayout.setShowMode(SwipeLayout.ShowMode.PullOut);
			viewHolder.swipeLayout.addDrag(SwipeLayout.DragEdge.Right, viewHolder.swipeLayout.findViewWithTag("Bottom2"));
			swipeLayouts.add(viewHolder.swipeLayout);
			viewHolder.iv_star= (ImageView) convertView.findViewById(R.id.star);
			viewHolder.iv_trash= (ImageView) convertView.findViewById(R.id.trash);
			convertView.setTag(viewHolder);
		}

		viewHolder.iv_star.setOnClickListener(new View.OnClickListener() {
			@Override
			public void onClick(final View v) {
				···
				//do what you like				
			}
		});
		viewHolder.iv_trash.setOnClickListener(new View.OnClickListener() {
			@Override
			public void onClick(View v) {
				···
				//do what you like
			}
		});
		viewHolder.twoStatusTime.setText(model.title);
		convertView.findViewById(R.id.item_surface).setOnClickListener(new View.OnClickListener() {
			@Override
			public void onClick(View v) {
				···
				//do what you like
			}
		});

		return convertView;
	}

	@Override
	public boolean isChildSelectable(int groupPosition, int childPosition) {
		// TODO Auto-generated method stub
		return true;
	}

	private class GroupViewHolder {
		ImageView groupImg;
		TextView groupName;
	}

	private class ChildViewHolder {
		public SwipeLayout swipeLayout;
		public ImageView iv_star,iv_trash;
		public ImageView card_type;
		public TextView twoStatusTime;
	}
	···
```

###child_status_item.xml

```java
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <com.daimajia.swipe.SwipeLayout
        android:id="@+id/sample"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:layout_marginLeft="15dp">

        <LinearLayout
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:tag="Bottom2">

            <ImageView
                android:id="@+id/star"
                android:layout_width="70dp"
                android:layout_height="60dp"
                android:background="@color/unStar"
                android:paddingLeft="22dp"
                android:paddingRight="22dp"
                android:src="@mipmap/ic_star_border_white_48dp" />

            <ImageView
                android:id="@+id/trash"
                android:layout_width="70dp"
                android:layout_height="60dp"
                android:background="@drawable/red"
                android:paddingLeft="25dp"
                android:paddingRight="25dp"
                android:src="@mipmap/trash" />
        </LinearLayout>

        <LinearLayout
            android:id="@+id/item_surface"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:background="#f7f7f7"

            android:gravity="center_vertical"
            android:orientation="horizontal">

            <ImageView
                android:id="@+id/iv_type"
                android:layout_width="40dp"
                android:layout_height="40dp"
                android:layout_marginLeft="30dp"
                android:layout_marginBottom="6dp"
                android:layout_marginRight="6dp"
                android:layout_marginTop="6dp"
                android:scaleType="centerInside" />

            <TextView
                android:id="@+id/two_complete_time"
                android:layout_width="0dp"
                android:layout_height="wrap_content"
                android:layout_weight="1"
                android:singleLine="true"
                android:ellipsize="end"
                android:textColor="#999999" />

        </LinearLayout>
    </com.daimajia.swipe.SwipeLayout>
</LinearLayout>

```
最后效果如下

<div  align="center">  
 <img style="box-shadow:5px 0 5px;" src="{{ site.url }}/assets/img/swipe.jpg" alt="swipe">
</div>


----------

[github](https://github.com/vienan/TimeLine "github")