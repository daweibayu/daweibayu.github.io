---
layout: post
title:  "设计模式漫谈"
author: "daweibayu"
tags: Android
excerpt_separator: <!--more-->
---
 <!--more-->

开门见山一句话，我认为设计模式的核心就是“”，说糙一点就是去除重复代码
用古早软件工程的话语体系说就是“低耦合，高内聚”；
用糙一点的话说就是既不要重复代码，又要好扩展；


工厂模式的核心是解除了对象创建导致的具体依赖，因为在对象的传递过程中可以使用父类型，但是在对象创建时一定是要依赖到特定类型的。
桥接模式的核心是避免多维度造成的子类数量指数级的膨胀。
单例模式就是避免创建重复对象并使其方便分享。
Builder模式的核心是避免初始化是构造函数参数过多。



现有的设计模式是基于面向对象逻辑的，

 常说组合优于继承，其实是因为继承其实也是一种耦合，子类与父类的耦合。而组合可以解除这种耦合。


 以 Android 中的列表举例
```kotlin
class CustomAdapter(private val dataSet: Array<String>) :
        RecyclerView.Adapter<CustomAdapter.ViewHolder>() {

    /**
     * Provide a reference to the type of views that you are using
     * (custom ViewHolder).
     */
    class ViewHolder(view: View) : RecyclerView.ViewHolder(view) {
        val textView: TextView

        init {
            // Define click listener for the ViewHolder's View.
            textView = view.findViewById(R.id.textView)
        }
    }

    // Create new views (invoked by the layout manager)
    override fun onCreateViewHolder(viewGroup: ViewGroup, viewType: Int): ViewHolder {
        // Create a new view, which defines the UI of the list item
        val view = LayoutInflater.from(viewGroup.context)
                .inflate(R.layout.text_row_item, viewGroup, false)

        return ViewHolder(view)
    }

    // Replace the contents of a view (invoked by the layout manager)
    override fun onBindViewHolder(viewHolder: ViewHolder, position: Int) {

        // Get element from your dataset at this position and replace the
        // contents of the view with that element
        viewHolder.textView.text = dataSet[position]
    }

    // Return the size of your dataset (invoked by the layout manager)
    override fun getItemCount() = dataSet.size

}
```

其实想一想，整个页面有独特的就三部分
1. 请求 api，https://base_url/version/target，这里其实只用 target，都不需要后边的分页等参数，因为分页逻辑本来就应该是统一的
2. 返回的数据
```json
{
  "results": [
    {
      "id": "1",
      "others": "aaaa"
    },
    {
      "id": "2",
      "others": "xxxx"
    }
  ]
}
```
```java
data class DataItem(val id: String, val others: String, ...)
```
3. UI，在 Android 中具体就是指 ViewHolder
```java
class DataItemHolder(context: Context, root: ViewGroup?) : BaseViewHolder<DataItem>(context, root, R.layout.layout_data_item) {

    override fun bindData(item: DataItem) {
        binding.idView.text = item.id
        binding.othersView.setContent(item.others)
    }
}
```

那最终呈现应该就是
```
class DataListFragment : BaseListFragment<FragmentContactListBinding>() {

    private val adapter = object : TypedCommonListAdapter<UserBean>() {

    }

        init {
                registerHolder(ContactViewHolder::class.java)
        }

    override fun initView() {
        val linearLayoutManager = LinearLayoutManager(requireContext())
        linearLayoutManager.orientation = RecyclerView.VERTICAL
        binding.recyclerView.layoutManager = linearLayoutManager
        binding.recyclerView.adapter = adapter
        binding.swipeRefresh.setOnRefreshListener(::refreshContact)
        refreshContact()
    }

    private fun refreshContact() {
        binding.swipeRefresh.isRefreshing = false
        binding.errorTip.gone()
        MainScope().launch {
            val result = RestApiClient.youpengService().getContact()
            when(result) {
                is ApiResponse.Ok -> {
                    adapter.addDataList(result.data.filter { it.objectId != UserManager.getCurrentUser()?.objectId })
                    adapter.notifyDataSetChanged()
                }
                else -> {
                    binding.errorTip.visible()
                }
            }
        }
    }
}
```
正常开发中，一定是前后端协同、rd pm 协同、产研与运营销售等部门协同，我一直认为好的业务（或者说“正确”的产品逻辑）一定可以通过