---
layout: post
title:  "设计模式漫谈"
author: "daweibayu"
tags: 设计模式
excerpt_separator: <!--more-->
---
 <!--more-->

开门见山一句话，我认为设计模式的核心就是“封装变化点”，用古早软件工程的话语体系说就是“低耦合，高内聚”，用糙一点的话说就是既不要重复代码，又要好扩展。

比如：

* 工厂模式的核心是解除了对象创建导致的具体依赖，因为在对象的传递过程中可以使用父类型，但是在对象创建时一定是要依赖到特定类型的；
* 桥接模式的核心是避免多维度造成的子类数量指数级的膨胀；
* 单例模式就是避免创建重复对象并使其方便分享；
* Builder模式的核心是避免初始化是构造函数参数过多；
* 常说组合优于继承，其实是因为继承其实也是一种耦合，子类与父类的耦合。而组合可以解除这种耦合
* ...

这些东西吧，就是理解的人不用多说，不理解的人多说也没用。我是一个`极度`不擅长记忆的人，尤其年龄大了，n 年前看过的设计模式，早忘的一干二净了。所以基本当别人扯到设计模式的时候，我一般都敬而远之，因为很多时候别人说一个设计模式的时候，我都不记得这个设计模式到底是干嘛用的了。这里还刚妄言设计模式，纯粹是想表述一下我对设计的理解与实践，话不多说，直接 show code。

## 举例

 以 Android 中的列表举例，我们可以先把元素列出，看看哪些属于模版代码

1. url
2. 返回的数据结构
3. 网络请求框架
4. 列表展示的 UI
5. 分页逻辑
6. 下拉刷新
7. 网络请求失败展示错误提示
8. 列表条目点击事件处理

暂时只列这几个比较公共的逻辑，我们可以挨个分析一下这些元素哪些是“公共”的，哪些是独有的

### url

以 restful api 为例，格式为 `${domain}/${version}/${targetObj}?offset=${offsetNum}&limit=${limitNum}`
我们可以看到其实其中的五个参数，只有 `${targetObj}` 是与本次业务相关，其他都是公共代码

* 推荐使用 restful api
* 客户端与服务器端在定 api 时一定要慎之又慎，可以简单理解为客户端与服务器端交互就是通过 api 的，api 设计的合理则前后端解藕。后续不论是前端重构还是后端重构就会互不影响。如果业务耦合，那前后端动代码都要互相同步，这样的后果不用多说，就是大家深陷泥潭，动身不得。

### 返回的数据结构

返回的数据如下：
```json
{
  "errorCode": "",
  "errorMsg": "",
  "results": [
    {
      "id": "xxxxxx1",
      ...
    },
    {
      "id": "xxxxxx2",
      ...
    }
  ]
}
```

其中 `errorCode、errorMsg、results` 也都是样式代码，都可以通过 json 解析一次性解决问题，只有 `results` 中的数据是不同的

在 kotlin 中基本都是一行代码解决问题：

```java
data class DataAItem(val id: String, val others: String, ...)
```

### 网络请求框架

这个不多说，Android 端现在的最佳实践就是 retrofit + okhttp

### 列表展示的 UI

在 Android 中，可以简单理解为单条目的 UI 对应的其实就是 holder

```java
class DataAItemHolder(context: Context, root: ViewGroup) : BaseViewHolder<DataAItem>(context, root, R.layout.layout_data_a_item) {

    override fun bindData(item: DataItem) {
        binding.idView.text = item.id
        binding.othersView.setContent(item.others)

        binding.idView.setOnClickListener {
            context.startActivity(...)
        }
    }
}
```

为了篇幅，这里就不列 R.layout.layout_data_a_item 了，相信 Androider 都明白

### 分页逻辑、下拉刷新、网络请求失败展示错误提示

与 url 结合，只要当时约定的 api 是格式化的，那么这里的分页逻辑与下拉刷新其实也都是公共的，因为所有类似列表页面的形式都是相同的
至于错误展示，基本逻辑也都是公共的

不同点：

* 部分页面不需要分页和下拉刷新
* 下拉刷新根据内容不同动画效果不同
* 网络请求失败根据内容不同展示提示不同

我们最后说这些问题的处理

### 列表条目点击事件处理

这个是根据内容不同事件是不同的，但是这部分的逻辑是可以些在 DataItemAHolder 中的，见上文中定义的 DataItemAHolder

### 理想完整形态

所以一个单独的列表页面理论上的所有代码便如下：

```java
data class DataAItem(val id: String, val others: String, ...)

class DataAItemHolder(context: Context, root: ViewGroup) : BaseViewHolder<DataAItem>(context, root, R.layout.layout_data_a_item) { ... }

class DataAListFragment : BaseListFragment<CommonListBinding>() {

    init {
        setPageUrlTarget("${targetObj}");
        registerHolder(DataAItemHolder::class.java)
    }
}

// 如果用注解形式，则更简洁
@Endpoint("targetObj")
@Holder(DataAItemHolder::class.java)
class DataAListFragment : BaseListFragment<CommonListBinding>() {}
```

还有一个 `layout_data_a_item.xml`

所有`变化点`都在代码里了，以这种形式去实现一个列表，就只有 `layout_data_a_item.xml` 会稍微费点时间，总共加起来也不会超过 1 小时，而且逻辑清晰、代码简洁、便于维护。

不需要 adapter，不需要 LayoutManager，不需要 ItemDecoration，甚至，这个 DataAListFragment.kt 都是模版代码，既然是模版代码，那就可以动态生成。
哪怕上述代码只能够替代 50% 的真实列表需求，其实都是极大的劳动力的解放。

至于为什么是`理想完整形态`，是因为我也`没有完全实现上述逻辑`，主要是以往写 sdk 居多，少写 UI，以上逻辑都是在我大概五六年前写过的一个框架的基础上优化而来。

### 问题

上边看着舒服，但是其实问题还是很多的。如果所有逻辑都往 `base` 或者 `common` 中塞，不用我多说，大家也知道是`垃圾设计`。  
我们做到了`不要重复代码`，那`好扩展`怎么办呢？  
像上边 `分页逻辑、下拉刷新、网络请求失败展示错误提示` 中所述的不同点，还有其他的：

* 部分页面不需要分页和下拉刷新
* 下拉刷新根据内容不同动画效果不同
* 网络请求失败根据内容不同展示提示不同
* 自定义 LayoutManager
* 自定义 ItemDecoration
* 自定义 adapter
* DataAItemHolder 如何创建，即 registerHolder 到底如何实现（在一个模版代码中创建具体类）
* 支持数据缓存
* ...


我们拿其中的几个举例：

#### 分页开关

在 `BaseBindingFragment` 中:

```java
fun enablePaging(): Boolean {
    return true
}
```

这就是简单的模版模式。如果 DataAListFragment 是动态生成的，那可以使用 Builder 模式。

#### Holder 如何创建

这里其实出现了反向依赖。正常来讲，如果要创建具体的 DataAItemHolder，那么模版代码一定要依赖 DataAItemHolder，不然没法调用构造函数。
这里解决方案是固定构造函数：

```java
  class DataAItemHolder(context: Context, root: ViewGroup) : BaseViewHolder<DataAItem>(context, root, R.layout.layout_data_a_item) {}
  ```

  即所有 holder 的构造函数都是固定的 `context: Context, root: ViewGroup`，那可以在 Adapter 中  

  ```java
  override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): CommonViewHolder<T> {
      holderTypeMap[viewType]?.let {
          val holder = it.getConstructor(Context::class.java, ViewGroup::class.java).newInstance(parent.context, parent)
          holder.setOnClickListener(viewHolderClickListener)
          return holder
      }
      throw IllegalArgumentException("The type :$viewType create exception")
  }
  ```

  通过这种形式创建，这里的 holderTypeMap 可以理解成 `DataAItem` 的缓存，即去看一下是否已经注册过该 Holder 了，因为 DataAItem 与 DataAItemHolder 是 1v1 绑定的关系。

当然还有其他解决方案，不过我个人目前感觉这应该算是比较好的。

#### 其他

其他问题怎么解决大家可以自己想办法，这里不做赘述。

## 大总结

以上使用了哪些设计模式？我也不知道。其实只要知道了 `理想完整形态`，那剩下的就是想办法去解决具体细节问题了，这些细节工作量占 80%，但是从设计角度讲大概只占 20%。不管怎么搞，只要能完成`既不要重复代码，又要好扩展`的目标，其实就不用管啥设计模式了。重意不重形。


## 番外篇

我这些年大多是做 sdk 开发，呆过的公司不少，亲眼见证了不少公司从 native -> h5 -> RN -> native -> flutter（or kmm） 的技术路线修改，五味杂陈。变得只是技术路线，写代码的依然还是`3年+ 初级工程师`。  
再一个例子，前公司为了增效专门聘请了一个敏捷教练，这其实与换技术路线如出一辙。这就像是拿着设计模式往里套，套不进去再换一个。

大部分人不是尽力去解决问题，而是把时光和精力花在绕过问题上。换技术路线并不能解决`产品逻辑问题`，也不能解决`3年+ 初级工程师`的问题。这两个问题才是核心。
产品逻辑问题由人去解决，3年+ 初级工程师的问题也是人的问题。而人的问题要从`内`解决，而不应该从外部下手。所谓的`内`无非也是两个，一是`能力`，二是`责任心`。

### 责任心问题

正常开发中，一定是前后端协同、rd pm 协同、产研与运营销售等部门协同，我一直认为好的业务（产品的`理想完整形态`）一定可以引导出代码上的合理架构。如果代码架构乱七八糟，原因无非两个，`一是产品逻辑问题，二是程序员能力问题`。这种通过代码设计过程中体现出的问题，绝大多数都可以追溯到产品逻辑上。这是产品优化的及其重要的一条渠道，但是遗憾的大多数时候，这条渠道名存实亡。原因无非也是两个，`一是程序员的责任心问题、二是 pm 的责任心问题`，大多数人本着能少一事就少一事的原则混饭吃，放任不合理的产品设计，自己也写不负责任的代码。当然`万方有罪，罪在朕躬`。

我是亲眼见过运维的同学在月度总结会上把影响公司营收10%以上的事故当成笑话讲，我也亲眼见过只做营销活动而一点不关心产品的事业部总监。其实我想说，程式化对应员工的公司，一定也会收获程式化应对的员工，这就是`你糊弄我，我糊弄你`，最终双输的局面，这其实是我离职这么多家公司的最核心原因。

### 能力问题

程序员更应该增加的对产品和业务的感知能力，产品、迭代流程、管理，其实都是可重构的，核心从来都不是设计模式，而是找到`理想完整形态`并落实。只有锻炼审美能力，才能知道代码的丑、产品的丑以及管理的丑。


关于二者的解决方案其实很简单，就是`人为本`。公司中其实是员工占主体，但管理层是大脑。管理层建立正向循环机制，逐步剔除混日子员工。你要是还问我怎么建立`正向循环`机制，那你是没理解`人为本`。  



闲庭随笔，大话漫谈，鄙俚浅陋，诸君勿怪。
