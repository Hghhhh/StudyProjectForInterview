# 1.项目及项目框架介绍

    git checkout step-1-project-framework

本项目基于webmagic开源项目制作一个优雅高效的爬虫。最终以豆瓣图书为例，看看怎么一步步完善我们的爬虫。

*--注意--*

**本项目仅作为技术学习研究使用，禁止用于任何商业用途，禁止任何损害网站利益的行为**

## webmagic
首先介绍一下webmagic开源项目。

> 源码：https://github.com/code4craft/webmagic

> 官网：http://webmagic.io/

比较遗憾的是webmagic项目已经没有维护了，但是这不妨碍我们的学习，因为webmagic本身就已经足够完善了，
而且其扩展性极好，是我们用java搭建爬虫的不二之选。

### PageProcessor
在webmagic中，你最需要关心的是` PageProcessor `接口，该接口中的` public void process(Page page); `方法是爬虫的核心方法，
你可以看到这个方法中传入了一个` Page `参数，` Page `封装了一次爬取的所有信息，包括页面信息和请求信息。
你需要做的是实现该接口，并将如何处理页面以及爬虫如何进行下一步迭代爬取的逻辑写入` public void process(Page page); `方法中。
webmagic会持有一个你实现了` PageProcessor `接口的类的引用，并在webmagic封装的多线程环境下循环调用` process(Page page);`方法。
在` process(Page page);`中，你可以使用` css、regex、xpath `三种方式来获取页面中你想要的元素（你甚至可以用‘$’符号去选取DOM）。
这里面xpath使用的是作者基于Jsoup自己实现的Xsoup，也是一个很棒的项目。

``` java
//示例如下：
public class GithubRepoPageProcessor implements PageProcessor {

    // 部分一：抓取网站的相关配置，包括编码、抓取间隔、重试次数等
    private Site site = Site.me().setRetryTimes(3).setSleepTime(1000);

    @Override
    // process是定制爬虫逻辑的核心接口，在这里编写抽取逻辑
    public void process(Page page) {
        // 部分二：定义如何抽取页面信息，并保存下来
        page.putField("author", page.getUrl().regex("https://github\\.com/(\\w+)/.*").toString());
        page.putField("name", page.getHtml().xpath("//h1[@class='entry-title public']/strong/a/text()").toString());
        if (page.getResultItems().get("name") == null) {
            //skip this page
            page.setSkip(true);
        }
        page.putField("readme", page.getHtml().xpath("//div[@id='readme']/tidyText()"));

        // 部分三：从页面发现后续的url地址来抓取
        page.addTargetRequests(page.getHtml().links().regex("(https://github\\.com/[\\w\\-]+/[\\w\\-]+)").all());
    }

    @Override
    public Site getSite() {
        return site;
    }
}
```
你可以调用` page.putField() `将你获取的结果存起来，` page `会持有一个` ResultItems `引用，
用来存放爬取过程中的结果。调用` page.addTargetRequest() `可以存入一个url作为下一个爬取的页面地址。

到这里你应该知道了，你的爬虫主要逻辑应该放在哪里了。

### Pipeline

` Pipeline `是又一常用接口，它帮你处理了你在` PageProcessor `中放入结果。
但是这里有一个问题：既然在` PageProcessor `的方法` process(Page page);` 中已经能够获取到所有的结果了，
为什么非要先存放起来，交由一个专门的` Pipeline `去处理呢？这里引用一段官方的解释：

> 可以看到，Pipeline其实就是将PageProcessor抽取的结果，继续进行了处理的，其实在Pipeline中完成的功能，你基本上也可以直接在PageProcessor实现，那么为什么会有Pipeline？有几个原因：

> 1.为了模块分离。“页面抽取”和“后处理、持久化”是爬虫的两个阶段，将其分离开来，一个是代码结构比较清晰，另一个是以后也可能将其处理过程分开，分开在独立的线程以至于不同的机器执行。

> 2.Pipeline的功能比较固定，更容易做成通用组件。每个页面的抽取方式千变万化，但是后续处理方式则比较固定，例如保存到文件、保存到数据库这种操作，这些对所有页面都是通用的。WebMagic中就已经提供了控制台输出、保存到文件、保存为JSON格式的文件几种通用的Pipeline。

你可以设置很多自定义的Pipeline，webmagic会在每次完成一个页面的爬取工作后，按顺序调用所有注册的Pipeline。
``` java
    private void onDownloadSuccess(Request request, Page page) {
        if (site.getAcceptStatCode().contains(page.getStatusCode())){
            pageProcessor.process(page);
            extractAndAddRequests(page, spawnUrl);
            if (!page.getResultItems().isSkip()) {
            //这里循环调用了所有注册的Pipeline
                for (Pipeline pipeline : pipelines) {
                    pipeline.process(page.getResultItems(), this);
                }
            }
        } else {
            logger.info("page status code error, page {} , code: {}", request.getUrl(), page.getStatusCode());
        }
        sleep(site.getSleepTime());
        return;
    }
```
在代码里面我已经实现了两个默认实现，结合上面的介绍相信你很容易就能看懂。
我们之后会实现自己的` Pipeline `以及对webmagic框架的扩展。

------
` PageProcessor ` 和 ` Pipeline `是我们需要重点关注的两个接口，当然webmagic还提供了各种爬虫需要的功能，
比如代理、定时任务、JSON格式解析（针对Ajax请求）等等常用功能，交给你去探索啦！

## 项目框架
如果你查看` pom.xml `文件，你一定会一目了然：我们主要使用了Spring Boot和MyBatis作为我们的主要框架。

我们预先设置了部分文件夹：
* ` dao ` 用来负责与数据库打交道
* ` model ` 数据模型都在这里
* ` service ` 爬虫的主体逻辑都在这里
* ` utils ` 各种工具类

是不是很简单！

### 设计模式
这里我们需要谈谈设计模式。

不知道你有没有发现一个问题：爬取一个页面、分析一个页面、将结果存下、寻找下一个迭代页面...甚至会包含不同层次的页面爬取...
这么多功能我们全部需要写进` PageProcessor `的` process(Page page);`方法中，
是不是想想就会觉得非常沉重，不仅沉重臃肿，而且没有可扩展性。这绝对不是优雅的解决方案。

这里我们需要先介绍设计模式中的其中一条原则：

> 1.面向接口编程，不要面向具体实现编程。

这里，我们将所有的逻辑堆砌在一个函数中，就是面向实现编程，我们太着急的去实现一个功能，
以至于将java写成了c语言。你可以想想要怎么做！

但是webmagic只提供这一个方法，我们要怎么扩展呢？接下来我们会演示。

在此之前，你需要了解至少三个设计模式

* 策略模式
* 装饰器模式
* 观察者模式

# 好了！我们开始吧！

    git checkout step-2-framework-extension

# 第二步：扩展框架

    git checkout step-2-framework-extension

在我们真正的去实现具体的爬虫功能之前，我们需要先对原有的框架做拓展。
这里，我们将全部注意力集中在如何将` PageProcessor `接口中的` process(Page page);`方法从具体实现中剥离出来。

## 策略模式
我们在service.spider包下新增了一个strategy包，用来存放我们所有的策略。

首先是我们的策略总接口：

``` java
public interface ProcessStrategy {

    /**
     * 爬虫具体执行的方法 {@link Page}
     */
    void doProcess(Page page);

}
```
结合` DefaultSpider `中的代码分析：

``` java
public class DefaultSpider implements PageProcessor {

    protected ProcessStrategy processStrategy;

    public void setProcessStrategy(ProcessStrategy processStrategy) {
        this.processStrategy = processStrategy;
    }
    
    ...
    
    /*--- 实现PageProcessor中的方法 ---*/
    @Override
    public void process(Page page) {
    
        if(processStrategy == null)
            throw new NullPointerException();
    
        preProcess(page);
        processStrategy.doProcess(page);
        afterProcess(page);
    }
    
    ...
    
}
```
这里清晰的看到原先的爬虫主体  DefaultSpider  实现了 process(Page page) 方法。
但是这个方法里面却什么事情都没有干，而是通过持有一个 ` ProcessStrategy `的实例，
通过调用 ` processStrategy.doProcess(page); `方法去实际实现爬虫的逻辑。
这里我们不关心具体交给了哪个processStrategy，我们只知道process()方法会被webmagic框架调用就可以了，
当webmagic框架调用这里的process()方法时，实际上被我们“移花接木”的交给了processStrategy去做。
而` ProcessStrategy `是一个接口，是可以在运行时被替换的。这样的设计给我们带了很大的操作空间。

还记得我们上一步讲的吗？ **面向接口编程而不要面向实现编程**，体会一下。

我们可以继续观察DefaultSpider，` processStrategy.doProcess(page); `执行前后都被 ` preProcess(page); ` 和 ` afterProcess(page); `方法包围，
仔细看这两个方法：

``` java
    protected void preProcess(Page page) {
    }
    
    protected void afterProcess(Page page) {
    }
```
这两个方法什么都没有干。这是为了扩展留下一条后路，如果你完成了爬虫逻辑之后，
你的同事需要在爬虫开始的时候，更新数据库中的一条数据，并且需要在爬完之后记录一条日志，
可是你有没有实现这些功能，怎么办呢？

你的同事只需要继承DefaultSpider类，然后自己重写上面这两个方法，就可以了。

这样做的好处就是可以不用修改你原先的爬虫逻辑，免得改出bug了不是？

这其实是设计模式中的另一条原则：

> 开闭原则：代码对扩展开放，对修改关闭。

实际上这里的设计参考了Spring源码，请看：

``` java
public class DefaultBeanDefinitionDocumentReader implements BeanDefinitionDocumentReader {

    ...
    
    /**
	 * Register each bean definition within the given root {@code <beans/>} element.
	 */
	@SuppressWarnings("deprecation")  // for Environment.acceptsProfiles(String...)
	protected void doRegisterBeanDefinitions(Element root) {
		// Any nested <beans> elements will cause recursion in this method. In
		// order to propagate and preserve <beans> default-* attributes correctly,
		// keep track of the current (parent) delegate, which may be null. Create
		// the new (child) delegate with a reference to the parent for fallback purposes,
		// then ultimately reset this.delegate back to its original (parent) reference.
		// this behavior emulates a stack of delegates without actually necessitating one.
		BeanDefinitionParserDelegate parent = this.delegate;
		this.delegate = createDelegate(getReaderContext(), root, parent);

		if (this.delegate.isDefaultNamespace(root)) {
			String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
			if (StringUtils.hasText(profileSpec)) {
				String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
						profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
				// We cannot use Profiles.of(...) since profile expressions are not supported
				// in XML config. See SPR-12458 for details.
				if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
					if (logger.isDebugEnabled()) {
						logger.debug("Skipped XML bean definition file due to specified profiles [" + profileSpec +
								"] not matching: " + getReaderContext().getResource());
					}
					return;
				}
			}
		}

        //在这里，看到了吗？
		preProcessXml(root);
		parseBeanDefinitions(root, this.delegate);
		postProcessXml(root);

		this.delegate = parent;
	}
	
	...
	
}
```

最后，对DefaultSpider做一下功能的封装，包括爬虫的一些设置，以及返回爬虫实例。

我们自己实现了一个类去实现ProcessStrategy接口：` OneBookProcessor.java`，
从名字你应该能知道，这个类的功能就是用来分析一个豆瓣图书页面的图书信息。
类内部提供了一个缓存以供之后使用（如果对这里有疑惑也可以先不加）。

这一步可能需要仔细理解一下，之后我们会慢慢看到这么做的好处。你需要自己实现这些步骤。

下一步，我们将具体实现如何对单个页面进行解析。

    git checkout step-3-onebook-implement

# 第三步：单个图书页面的爬取

    git checkout step-3-onebook-implement

我们的目标是将图书的名字，封面图片还有豆瓣评分，这三个数据爬取到。
前面我们说过，爬取到的数据可以通过调用
>   page.putField()

将结果存储起来，等待Pipeline去处理。这里给出一条建议：

>   如果需要处理的数据超过两条，那么将他们封装成一个model吧！

于是，我们创建了 OriginBook.java 类，用来封装图书数据。

我们随便打开一个豆瓣上的图书页面，分析它的网页结构，在这里不详细叙述如何分析一个html页面，
如果你对这一块有疑问，建议学习一下DOM元素的选取。

在分析完毕之后，相应的使用css或者xpath语法去抓取DOM元素就可以了。这里不再赘述css和xpath的具体语法，这不是我们的重点。
基本思路就是寻找有唯一表示的div，根据class一层层定位（最好直接根据id定位，因为id是唯一的），最后找到想要的元素和属性。
你一定要自己完成html的分析以及对你感兴趣的数据的提取工作。
主要的代码在这里：

``` java
public class OneBookProcessor implements ProcessStrategy {

    ...
    
    protected String parseName(Page page) {
        return page.getHtml().xpath("//div[@id='wrapper']/h1/span/text()").toString();
    }
    
    protected String parseImg(Page page) {
        return page.getHtml().xpath("//div[@id='mainpic']/a/img/@src").toString();
    }
    
    protected int parseScore(Page page) {
        try {
            String score = page.getHtml().xpath(
              "//div[@id='interest_sectl']//strong[contains(@class, 'll') and contains(@class, 'rating_num')]/text()")
              .toString();
            score = score.replaceAll("[\\s|\\u00A0.]", "");
            return Integer.valueOf(score);
        } catch (Exception e) {
            return -1;
        }
    }
    
    ...
    
```
### CallablePipeline.java
我们自定义了一个 CallablePipeline.java ，用来返回一个 Object 类型的结果。

考虑到：有些逻辑不适合写到 Pipeline 中，
而且 Pipeline 不一定会完全将结果消费掉，这时我们会希望得到 Pipeline 中的结果，然后对结果单独处理。
webmagic认为我们应该在 Pipeline 中完全消费掉结果（存到数据库，或者存到文件等等），
但是很多时候我们无法做到，比如我们期待把这个结果返回到一个页面，
由一个工作人员确认爬取的信息无误，再将结果插入数据库。所以我们让 CallablePipeline 继承 Pipeline ，
并扩展了一个 getResult 方法，相当于告诉 Pipeline ：我把结果取出来交给别人，你去忙你的吧！。这样，我们可以随时从 Pipeline 中获取一个结果。

但是这里有更加优雅的做法！我们之后会展示。

现在我们调用 SpiderService 中的` getLotsOfBooks() `，传入一个豆瓣图书的url，就可以获得一个爬取到的书籍数据了！是不是很棒！

### 阶段总结：
这一步里，使用到了**策略模式**，我们将具体爬取数据的逻辑，与上层调用的逻辑，比较彻底的解耦了。SpiderService 持有了一个 DefaultSpider，
而DefaultSpider里面没有具体的实现，具体的实现由 OneBookProcessor 完成。OneBookProcessor 专注于书籍页面的解析算法，
DefaultSpider 负责使用这个算法，并设置爬虫的各种性质。如果你的同事有了新的更快更高效的书籍信息提取算法XXXProcessor，
只要实现了 ProcessStrategy ，就可以很方便的替换掉DefaultSpider里面的算法，而 DefaultSpider 一句代码也不用改。
当然，好处不仅仅只有这些。接下来你能看到。

我们希望所有的代码你都有亲自去实现，这是最关键的一步。

下一步，我们将实现爬虫的迭代爬取功能：

    git checkout step-4-books-iterator

# 第四步：图书的迭代爬取

    git checkout step-4-books-iterator

在前面几步里面，我们详细讲解了如何利用**策略模式**，将 process(Page page) 方法的实现逻辑剥离出来。
这一步里面，我们将利用 **装饰器模式** 对功能进行扩展。

可是，一个爬虫往往不会单单去爬取一个页面，往往会根据一个页面或者某种规则去进行迭代爬取。现在，
我们已经实现了单个页面的爬取，那我们如何实现爬虫的迭代呢？在 OneBookProcessor 里面写一个循环呀！如果这句话是同学们的第一反应的话，
那你应该重新去体会：

> 面向接口编程，不要面向实现编程。以及，对修改关闭，对扩展开方

如果我们又去修改 OneBookProcessor 里面的代码，实在不能称得上优雅，因为 OneBookProcessor 已经完美的实现了它的使命--对单个图书页面的爬取，
我们没有任何理由去修改它--那样只能让你的代码越来越糟。所以你应该将你的目光从 OneBookProcessor 身上移开，去寻找其他的方法。

仔细思考就会发现，无论如何，爬虫最后一定会落实到 OneBookProcessor 身上，去对一本图书进行爬取，我们真正需要的是，
设计一种迭代策略，然后在迭代的最后一步调用 OneBookProcessor::doProcess(Page page)（"::"符号是java8的语法，这里指代一个方法）。

从我们的分析，可以看到，迭代的策略需要持有一个 OneBookProcessor 实例，用于调用真正爬取一本图书的方法。

## 装饰器模式（参考java.io包的实现）

首先我们需要一个伪实现类，该类主要的作用是作为一个标识，所有继承于这个类的都是装饰器。代码如下

``` java
    /**
     * FilterProcessor 是ProcessStrategy的装饰类，用来扩展ProcessStrategy的爬虫方法。
     * 这是一个伪实现类，真正的装饰类需要继承这个类。
     * 设计参考 {@link java.io.FilterInputStream}
     */
    public class FilterProcessor implements ProcessStrategy {
    
        protected volatile ProcessStrategy processStrategy;
        
        public FilterProcessor(ProcessStrategy processStrategy){
            this.processStrategy = processStrategy;
        }
        
        @Override
        public void doProcess(Page page) {
            processStrategy.doProcess(page);
        }
    }
```
可以看到，FilterProcessor 是一个伪实现，就是什么都没干，只是用于标识装饰器的。真正的装饰器类需要继承 FilterProcessor 并自己实现：

### 重头戏 IteratorProcessor

直接上代码
``` java 
/**
 * 装饰者，用来装饰book的迭代爬取功能
 */
public class IteratorProcessor extends FilterProcessor {
    
    private Set<String> requestCache = new HashSet<>(32);
    
    public IteratorProcessor() {
        super(new OneBookProcessor());
    }
    
    public IteratorProcessor(ProcessStrategy processStrategy) {
        super(processStrategy);
    }
    
    @Override
    public void doProcess(Page page) {
        if (StringUtils.startsWith(page.getRequest().getUrl(), "https://book.douban.com/tag/")) {
            requestCache.addAll(page.getHtml().regex("https://book.douban.com/subject/[0-9]+/?").all());
            page.addTargetRequests(new ArrayList<>(requestCache));
            requestCache.clear();
            page.setSkip(true);
        } else {
            processStrategy.doProcess(page);
        }
    }
}
```
首先，IteratorProcessor 提供了两个重载的构造方法，其中一个默认持有了一个 OneBookProcessor ，
如果你有其他的对于单页面的爬取更好的实现策略，可以使用第二个构造方法。

主体方法部分 doProcess 也非常好懂，首先我们根据当前爬取的页面类型判断，如果是图书的上一级页面，
我们就去分析页面里面含有的单本图书的url，单本图书的url的正则为

>   "https://book.douban.com/subject/[0-9]+/?"

图书的上一级一定是以"https://book.douban.com/tag/" 开头，这一点是我们之前分析页面结构得到的信息。
这样，我们将一个上级页面中所有的单本图书url放入了 TargetRequest 之中，webmagic回循环从 TargetRequest 中去取一个url去爬取。

这一步我们设置了 ` page.setSkip(true); `，意味着，这一步不需要经过 Pipeline 去处理。

如果不是上级页面（也就是单本图书页面，这里的逻辑并不严谨，但是不妨碍理解），
则调用 ` processStrategy.doProcess(page) ` ，也就是 OneBookProcessor 的处理逻辑 。
这里的逻辑比较复杂，比较难想清楚。建议类比于树的前序遍历去理解，将网页不同深度的url对应为一棵树，
最开始的入口页面为根节点，单个的图书页面的为叶子节点。

这样，我们就完成了对一个https://book.douban.com/tag/ 开头的页面的图书爬取。

当然，网页的结构可能有很多层，往往需要进入好几层，才能真正找到需要的数据页面，你可以延续这个思路，
在 IteratorProcessor 外面再加几层装饰，完成多层次的爬取工作，每一层只用专注于当前层次的网页的爬取和信息收集工作。
如果这层的信息（往往是多个下一层的url）收集完毕，则可以调用 ` page.addTargetRequests ` 方法将接下来需要迭代的url放入webmagic容器中，
之后就可以交给更深层的 ProcessorStrategy 去完成工作了。

总结：这样做的好处就是，扩展性非常的良好。每一层都只关心这一层的逻辑，只要这一层的实现完毕，就不用再操心这一层的逻辑了。

最后需要注意的一点就是，只要不想交给 Pipeline 的结果，都需要调用 ` page.setSkip(true) `。

` TargetRequests `内部实现实际上是一个队列，同学们需要考虑url的放入顺序问题。

到这里，我们单本图书的爬取，以及多本图书的迭代爬取都已经完成了！你需要自己去实现所有的逻辑，这点至关重要。

接下来你需要思考一个问题：

如果你的同事在很多其他的地方，需要用到Pipeline里面的结果，而且他们不愿意调用 Pipeline 的 getResult 方法（因为你的结果可能还没有准备好呢），
他们想要等你有结果之后，就主动的就通知他们。怎么办呢？适合用设计模式中的那种模式去解决呢？
如果你想到了解决办法，请自己实现。

下一步：我们将利用观察者模式扩展 Pipeline ，使得结果可以主动推送到需要结果的类。

    git checkout step-5-pipeline-extension

# 第五步：利用观察者模式对Pipeline进行扩展。

    git checkout step-5-pipeline-extension

前面已经介绍过了，Pipeline会在一个爬取任务完成后被调用。
对结果的处理逻辑相对简单的情形下（比如直接存入数据库，或者打印到控制台等，都认为是比较简单的逻辑），
我们可以将处理逻辑直接写进 Pipeline 中。但是在很多情况下，事情远没有这么简单。
有的时候，Pipeline中获得的结果只能作为初步的结果，需要进行进一步的加工、检错、包装等等工作，
在一个项目环境中，这个初步的结果很可能被很多其他的模块引用。

比如对于你爬取的数据 result，项目中的A、B、C三个模块都需要获取到 result ，
虽然在我们扩展的 CallablePipeline 中提供了一个获取结果的方法 getResult，
但是A、B、C并不知道什么时候去调用 getResult，因为 Pipeline 可能根本还没有将结果准备好。

一种比较好的思路就是当 Pipeline 把结果准备好的时候，主动去推送结果。顺着这个思路，
我们使用**观察者模式**对其进行扩展。

## 观察者模式

首先，我们准备的两个接口：

> Observable.java

``` java 
public interface Observable {
    //增加一个观察者
    void addObserver(Observer o);
    //删除一个观察者
    void deleteObserver(Observer o);
    //通知我们所有的观察者
    void notifyObservers(Object arg);
}
```

> Observer.java

``` java
/**
 * 参考{@link java.util.Observer}设计
 */
public interface Observer {
    //观察者被通知的方法
    void update(Observable o, Object arg);

}
```

### 同学们可以思考一下，Pipeline 还有 A、B、C三个模块谁是Observable、谁是Observer。

好了，毫无疑问 Pipeline 应该去实现 Observable 接口。

现在问题又来了，我们希望所有实现了 CallablePipeline 接口的 Pipeline 的类都是 Observable，
怎么做到呢？直接将 Observable 接口中的方法放进 CallablePipeline 中吗？这时你违反了设计模式中的什么原则呢？

java的接口是允许多继承的，我们直接让 CallablePipeline 继承 Observable 即可。
理解这里非常重要，你必须明白我们为什么必须让 CallablePipeline 继承 Observable，
而不是这么做：

* 让实现了 CallablePipeline 的类同时实现 Observable
* 将 Observable 设计为一个类，让需要 Observable 功能的类继承 Observable。
* 将 Observable 里面的方法全部放进 CallablePipeline。

### 以上三种方案为什么不可以呢？交给你去探索。

好了，我们在 OriginBookPipeline 里面实现 CallablePipeline 的所有方法：

``` java
@Service
public class OriginBookPipeline implements CallablePipeline {
    
    ...
    
    @Override
    public void process(ResultItems resultItems, Task task) {
        originBook = resultItems.get("book");
        originBook.setOriginUrl(task.getSite().getDomain());
        notifyObservers(getResult());
    }
    
    ...
    
    /*--  implements Observable  --*/
    private Vector<Observer> obs;
    
    @Override
    public synchronized void addObserver(Observer o) {
        if (o == null) {
            throw new NullPointerException();
        }
        if (!obs.contains(o)) {
            obs.addElement(o);
        }
    }
    
    @Override
    public synchronized void deleteObserver(Observer o) {
        obs.removeElement(o);
    }
    
    @Override
    public void notifyObservers(Object arg) {
        Object[] arrLocal;
    
        synchronized (this) {
            arrLocal = obs.toArray();
        }
        for (int i = arrLocal.length - 1; i >= 0; i--) {
            ((Observer) arrLocal[i]).update(this, arg);
        }
    }
}
```

这里实现了 Observable 的三个方法。并使用了 ` Vector<Observer> obs ` 保存所有的观察者。

并且在 process 方法中调用了 notifyObservers 方法，并将结果（originBook）作为参数传入其中。

这里还是有必要对 Observable 中被实现的三个方法做一下解释：

* ` addObserver ` 注册一个观察者，不能为空。这个方法是同步的。
* ` deleteObserver ` 删除一个观察者。这个方法是同步的。
* ` notifyObservers(Object arg) ` 这个方法是核心，当 notifyObservers 被触发后，通知所有注册的观察者。` arg `封装了通知的信息。

这样一来就很明白了，我们先实现两个简单的 Observer ，ObserverA 和 ObserverB。

``` java 
public class ObserverA implements Observer {

    @Override
    public void update(Observable o, Object arg) {
        if (o instanceof OriginBookPipeline)
            //TODO with arg
            System.out.println("ObserverA " + ((OriginBook) arg).getScore());
    }
}
```

然后在 SpiderService 中为 Pipeline 注册ObserverA 和 ObserverB。

``` java
    public void getLotsOfBooks(String beginUrl) {

        ...
    
        pipeline.addObserver(new ObserverA());
        pipeline.addObserver(new ObserverB());
        defaultSpider.setProcessStrategy(new IteratorProcessor());
        ...
        
    }
```

好了，运行一下你的程序吧！看看 ObserverA 和 ObserverB 是否被推送了正确的结果！

如果你的模块 C 也需要被推送结果，只需要让 C 像 A、B 一样实现 Observer 接口，
并通过` pipeline.addObserver(new ObserverC()) ` 进行注册。之后，一旦 Pipeline 里面有了新的结果，就会通知所有的观察者！

当然我们同时为 Pipeline 保留了 getResult 方法，如果某个模块认为注册成为 Observer 太复杂，
并且这个模块也知道合适的调用时机，那么可以直接调用 getResult 得到想要的结果数据。

题外话：JDK 提供了现成的观察者模式的类和接口，我们在具体实现上，也是直接参考的 JDK 的实现。

但是 JDK 将 Observable 直接封装成了一个类，而且没有实现任何接口。很显然，这是面向实现编程，而不是面向接口，是一个比较糟糕的设计。而且我们之前说过，如果一个类希望具有 Observable 的特性，那么它必须继承 Observable，这样做的代价太大，因为java是单继承。java.util.Observable更像是一个示例，而不是一个可以用的接口。所以我们这里重新设计了 Observable 接口。

好了！你已经完成了非常重要的学习，以上的代码请一定自己完成！让我们复习一下：

1.首先，我们在扩展框架和实现我们的功能的过程中，始终遵循着一条原则：不去修改已经完成的部分，也就是开闭原则。

2.然后，我们尽力去面向接口编程，因为这样会给你的程序带来极好的可扩展性和解耦。

3.我们在三个不同的地方使用了三种设计模式，演示了，如何对一个既有的框架进行优雅的扩展。

--------
在最后留下了一个问题，一个难题，需要你们自己去解决：我们为了将目光集中在，如何利用设计模式扩展webmagic框架上，所以始终以单线程设计和运行我们例子，并没有做任何多线程安全控制。多线程控制本身就是一个巨大的话题，这里无论如何没有空间去展开。如果你对此感兴趣的话，你可以自己去完成并发安全的部分！而且我强烈建议你那样去做。（我们以后会推出多线程相关的项目）
