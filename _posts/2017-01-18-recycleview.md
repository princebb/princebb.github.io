---
layout:     post
title:      "Recycleview初窥"
subtitle:   "Recycleview Introduce"
date:       2017-01-18
author:     "Allen"
header-img: "img/code-bg.jpg"
tags:
    - Android
---

先让我们来看看Google文档中是如何定义RecyclerView的：

A flexible view for providing a limited window into a large data set.

(能够在有限的窗口中展示大数据集合的灵活视图。)

所以我们能够理解为，RecyclerView一个恰当的使用场景是：由于尺寸限制，用户的设备不能一次性展现所有条目
，用户需要上下滚动以查看更多条目。滚出可见区域的条目将被回收，并在下一个条目可见的时候被复用。

### 与Lisview的相比

1. **Adapter中的ViewHolder模式**。对于ListView来说，通过创建ViewHolder来提升性能并不是必须的。 因为ListView并没有严格的ViewHolder设计模式。 但是在使用RecyclerView的时候，Adapter必须实现至少一个ViewHolder，必须遵循ViewHolder设计模式。

2. **自定义Item条目**。ListView只能实现垂直线性排列的列表视图，与之不同的是，RecyclerView可以通过设置RecyclerView.LayoutManager来定制不同风格的列表视图，比如水平滚动列表或者不规则的瀑布流列表。

3. **Item动画**。在ListView中没有提供任何方法或者接口，开发者难以实现Item的增删动画。 相反地，可以通过设置RecyclerView的RecyclerView.ItemAnimator来为条目增加动画效果。

4. **数据绑定**。在LisView中针对不同数据封装了各种类型的Adapter，比如用来处理数组的ArrayAdapter和用来展示Database结果的CursorAdapter。 相反地，在RecyclerView中必须自定义实现RecyclerView.Adapter并为并其提供数据集合。

5. **Item分割线**。在ListView中可以通过设置android:divider属性来为两个Item间设置分割线。 如果想为RecyclerView添加此效果，则必须使用RecyclerView.ItemDecoration，这种实现方式不仅更灵活，而且样式也更加丰富。

6. **点击Or触摸事件**。在ListView中存在AdapterView.OnItemClickListener接口，用来绑定条目的点击事件。但是，很遗憾的是在RecyclerView中，并没有提供这样的接口，不过，提供了另外一个接口RcyclerView.OnItemTouchListener，用来响应条目的触摸事件。


### RecyclerView.Adapter示例

```java
public class ItemAdapter extends RecyclerView.Adapter<ItemAdapter.ViewHolder>{
	private Context context;
	private List<String> item;

	public static ItemAdapter created(Context context,List<String> items){
		return new ItemAdapter(context,items);
	}

	private ItemAdapter(Context context,List<String> items){
		this.context = context;
		this.items = items;
	}

	@Override public ViewHolder onCreateViewHolder(ViewGroup parent,int viewType){
		returnnew ViewHolder(LayoutInflater.from(context).inflater(R.layout.text_item,parent,false));
	}

	@Override public void onBindViewHolder(ViewHolder holder,int position){
		holder.textView.setText(items.get(position));
	}

	@Override public int getItemCount(){
		return (this.items !=null)?items.size():0;
	}

	public static final class ViewHolder extends RecyclerView.ViewHolder{
		@NonNull @Bind(R.id.text) protected TextView textView;

		public ViewHolder(View itemView){
			super(itemView);
			ButterKinfi.bind(ViewHolder.this,itemView);
		}
	}
}
```

### RecyclerView.LayoutManager


- LinearLayoutManager 垂直或者水平的Item视图。
    ```java
LinearLayoutManager layoutManager = new LinearLayoutManager(this);
layoutManager.setOrientation(LinearLayoutManager.VERTICAL);//垂直
layoutManager.setOrientation(LinearLayoutManager.HORIZONTAL);//水平
recyclerView.setLayoutManager(layoutManager);
```

- GridLayoutManager 网格Item视图
    ```java
GridLayoutManager gridLayoutManager = new GridLayoutManager(this, 2);
recyclerView.setLayoutManager(gridLayoutManager);
```

- StaggeredGridLayoutManager 交错的网格Item视图(瀑布流)。
    ```java
StaggeredGridLayoutManager staggeredGridLayoutManager = new StaggeredGridLayoutManager(2,StaggeredGridLayoutManager.VERTICAL);
recyclerView.setLayoutManager(staggeredGridLayoutManager);
```

### LayoutManager工作原理
![](/img/recycleview-img/tree.jpg)
首先是 RecyclerView 继承关系，可以看到，与 ListView 不同，他是一个 ViewGroup。既然是一个 View，那么就不可少的要经历 onMeasure()、onLayout()、onDraw() 这三个方法。 实际上，RecyclerView 就是将 onMeasure()、onLayout() 交给了 LayoutManager 去处理，因此如果给 RecyclerView 设置不同的 LayoutManager 就可以达到不同的显示效果。

### RecyclerView 缓存与复用的原理
![](/img/recycleview-img/cache.jpg)
RecyclerView 的内部维护了一个二级缓存，滑出界面的 ViewHolder 会暂时放到 cache 结构中，而从 cache 结构中移除的 ViewHolder，则会放到一个叫做 RecycledViewPool 的循环缓存池中。
顺带一说，RecycledView 的性能并不比 ListView 要好多少，它最大的优势在于其扩展性。但是有一点，在 RecycledView 内部的这个第二级缓存池 RecycledViewPool 是可以被多个 RecyclerView 共用的，这一点比起直接缓存 View 的 ListView 就要高明了很多，但也正是因为需要被多个 RecyclerView 公用，所以我们的 ViewHolder 必须继承自同一个基类(即RecyclerView.ViewHolder)。
默认的情况下，cache 缓存 2 个 holder，RecycledViewPool 缓存 5 个 holder。对于二级缓存池中的 holder 对象，会根据 viewType 进行分类，不同类型的 viewType 之间互不影响。
