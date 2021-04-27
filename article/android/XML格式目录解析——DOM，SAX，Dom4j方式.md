### 一、概况

项目中要解析一个XML格式的目录，经过搜索了解到，解析方式主要有DOM，Pull，SAX三种方式，各自特点如下：

> **SAX**
>
> sax是一个用于处理xml事件驱动的“推”模型；
>
> 优点：解析速度快，占用内存少，它需要哪些数据再加载和解析哪些内容。
>
> 缺点：它不会记录标签的关系，而是需要应用程序自己处理，这样就会增加程序的负担。
>
> **DOM**
>
> dom是一种文档对象模型；
>
> 优点：dom可以以一种独立于平台和语言的方式访问和修改一个文档的内容和结构，dom技术使得用户页面可以动态的变化，如动态显示隐藏一个元素，改变它的属性，增加一个元素等，dom可以使页面的交互性大大增强。
>
> 缺点：dom解析xml文件时会将xml文件的所有内容以文档树方式存放在内存中。
>
> **PULL**
>
> pull和sax很相似，区别在于：pull读取xml文件后触发相应的事件调用方法返回的是数字，且pull可以在程序中控制，想解析到哪里就可以停止解析。 （SAX解析器的工作方式是自动将事件推入事件处理器进行处理，因此你不能控制事件的处理主动结束；而Pull解析器的工作方式为允许你的应用程序代码主动从解析器中获取事件，正因为是主动获取事件，因此可以在满足了需要的条件后不再获取事件，结束解析。pull是一个while循环，随时可以跳出，而sax不是，sax是只要解析了，就必须解析完成。）

所给的目录文件是这样的：

```xml
<?xml version="1.0" encoding="utf-8" ?>
<contents body-start-page="5">
    <item name="封面" start-page="1" end-page="1"></item>
    <item name="目录" start-page="3" end-page="3"></item>
    <item name="一、复习与提高" start-page="5" end-page="7">
        <item name="符号表示数" start-page="6" end-page="6"></item>
        <item name="小数" start-page="7" end-page="7"></item>
    </item>
    ...
    <item name="六、整理与提高" start-page="79" end-page="91">
        <item name="小数的四则混合运算" start-page="80" end-page="80"></item>
        ...
        <item name="数学广场——编码" start-page="91" end-page="91"></item>
    </item>
    <item name="说明" start-page="93" end-page="93"></item>
    <item name="封底" start-page="95" end-page="95"></item>
</contents>
```

### 二、解析

#### 1、先用DOM方式搞一搞：

```kotlin
fun getCatalogDOM(context: Context): ArrayList<Catalog> {
        val catalogs = ArrayList<Catalog>()
        try {
            val factory = DocumentBuilderFactory.newInstance()
            val builder = factory.newDocumentBuilder()
            val inputStream = context.resources.assets.open("contents.xml")
            val document = builder.parse(inputStream)
            val root = document.documentElement
            //以上获取到了根节点
            catalogs.addAll(parse(0, root))
        } catch (e: Exception) {
            e.printStackTrace()
        }
        return catalogs
    }

    private fun parse(level: Int, element: Element): ArrayList<Catalog> {
        val catalogs = ArrayList<Catalog>()
        val nodes = element.childNodes
        for (i in 0 until nodes.length) {
            val element2 = nodes.item(i)
            if (element2 is Element) {
                val catalog = Catalog()
                catalog.name = element2.getAttribute("name")
                catalog.startPage = element2.getAttribute("start-page")
                catalog.endPage = element2.getAttribute("end-page")
                catalog.level = level
                catalogs.add(catalog)
            }
            if ("item".equals(element2.nodeName) && element2.nodeType == Document.ELEMENT_NODE) {
                //还有子节点
                val newLevel = level + 1
                catalogs.addAll(parse(newLevel, element2 as Element))
            }
        }
        return catalogs
    }
```

把目录定义了不同的层级，用level表示。

```java
public class Catalog {

    private String name;
    private String startPage;
    private String endPage;
    private int level;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getStartPage() {
        return startPage;
    }

    public void setStartPage(String startPage) {
        this.startPage = startPage;
    }

    public String getEndPage() {
        return endPage;
    }

    public void setEndPage(String endPage) {
        this.endPage = endPage;
    }

    public int getLevel() {
        return level;
    }

    public void setLevel(int level) {
        this.level = level;
    }


    @Override
    public String toString() {
        return "Catalog{" +
                "name='" + name + '\'' +
                ", startPage='" + startPage + '\'' +
                ", endPage='" + endPage + '\'' +
                ", level=" + level +
                '}';
    }
}
```

上面已经说了，DOM解析会占用较多的内存，作为一个追求完美的人，这肯定是不能接受的，下面说说用SAX方式解析。

#### 2、SAX解析

```kotlin
fun getChapterSAX(context: Context): ArrayList<Chapter> {
        val chapters = ArrayList<Chapter>()
        try {
            val factory: SAXParserFactory = SAXParserFactory.newInstance()
            val parser: SAXParser = factory.newSAXParser()
            val xmlReader: XMLReader = parser.xmlReader
            val chapterHandler = ChapterHandler()
            xmlReader.contentHandler = chapterHandler
            xmlReader.parse(InputSource(context.resources.assets.open("contents.xml")))
            chapters.addAll(chapterHandler.chapters)
        } catch (e: Exception) {
            e.printStackTrace()
        }
        return chapters
    }
```

```java
public class ChapterHandler extends DefaultHandler {

    private Stack<Catalog> catalogStack = new Stack<>();
    private List<Chapter> chapters = new ArrayList<>();//有很多章
    private List<Catalog> catalogs = new ArrayList<>();//存放一章中的节，集合

    public List<Chapter> getChapters() {
        return chapters;
    }

    @Override
    public void startDocument() throws SAXException {
        super.startDocument();
        Log.e("www", "------startDocument------");
    }

    @Override
    public void endDocument() throws SAXException {
        super.endDocument();
        Log.e("www", "------endDocument------");
    }

    @Override
    public void startElement(String uri, String localName, String qName, Attributes attributes) throws SAXException {
        super.startElement(uri, localName, qName, attributes);
        Log.e("www", "startElement  uri=" + uri + ",localName=" + localName + ",qName=" + qName + ",attributes=" + attributes);
        if (TextUtils.equals("contents", qName))
            return;
        if (TextUtils.equals("item", qName)) {
            Catalog catalog = new Catalog();
            for (int i = 0; i < attributes.getLength(); i++) {
                if (TextUtils.equals("name", attributes.getQName(i))) {
                    catalog.setName(attributes.getValue(i));
                } else if (TextUtils.equals("start-page", attributes.getQName(i))) {
                    catalog.setStartPage(attributes.getValue(i));
                } else if (TextUtils.equals("end-page", attributes.getQName(i))) {
                    catalog.setEndPage(attributes.getValue(i));
                }
            }
            if (catalog != null) {
                catalogStack.push(catalog);
            }

        }

    }

    @Override
    public void endElement(String uri, String localName, String qName) throws SAXException {
        super.endElement(uri, localName, qName);
        Log.e("www", "endElement   uri=" + uri + ",localName=" + localName + ",qName=" + qName);
        if (TextUtils.equals("item", qName)) {
            if (!catalogStack.empty()) {
                Catalog catalog = catalogStack.pop();
                if (catalogStack.empty()) {
                    //队列空了，这一章结束.
                    Chapter curChapter = new Chapter();//当前章
                    curChapter.setName(catalog.getName());
                    curChapter.setStartPage(catalog.getStartPage());
                    curChapter.setEndPage(catalog.getEndPage());
                    curChapter.setCatalogs(catalogs);
                    catalogs.clear();
                    chapters.add(curChapter);
                } else {
                    catalogs.add(catalog);
                }
            }
        }
    }
}

```

SAX是按标签驱动的，不会记录标签之间的关系，开始解析的时候先回调startDocument() ,然后一个标签开始的时候会回调startElement()，标签结束的时候会回调endElement()，整个文件结束的时候回调endDocument()。这样各自对应关系很难找，所以我又定义了一个目录的类，目录里边有章节。

```java
public class Chapter {
    private String name;
    private String startPage;
    private String endPage;
    private List<Catalog> catalogs=new ArrayList<>();

    public String getStartPage() {
        return startPage;
    }

    public void setStartPage(String startPage) {
        this.startPage = startPage;
    }

    public String getEndPage() {
        return endPage;
    }

    public void setEndPage(String endPage) {
        this.endPage = endPage;
    }

    public List<Catalog> getCatalogs() {
        return catalogs;
    }

    public void setCatalogs(List<Catalog> catalogs) {
        if (catalogs != null)
            this.catalogs.addAll(catalogs);
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public String toString() {
        return "Chapter{" +
                "name='" + name + '\'' +
                ", startPage='" + startPage + '\'' +
                ", endPage='" + endPage + '\'' +
                ", catalogs=" + catalogs +
                '}';
    }
```

这样就可以解析出这个目录了。**但是**，如果某一天目录增加了层级，那我个方法还要改，也就是这个SAX解析不如DOM那种方法智能，这同样是不能接受的。经过查找资料，原来有[Dom4j](https://github-production-release-asset-2e65be.s3.amazonaws.com/38423426/12e1f600-7c39-11ea-934c-d567a8e33f39?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAIWNJYAX4CSVEH53A%2F20201209%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20201209T064430Z&X-Amz-Expires=300&X-Amz-Signature=2108625f46092c444370fb2b503d0ad5cde47466d228a1fbb9509d3c774549e3&X-Amz-SignedHeaders=host&actor_id=3131990&key_id=0&repo_id=38423426&response-content-disposition=attachment%3B%20filename%3Ddom4j-2.0.3.jar&response-content-type=application%2Foctet-stream)这个神器，这样解析就非常方便了。还是直接上代码：

```kotlin
fun getChapterSAX2(context: Context): List<Catalog> {
        val catalogs = ArrayList<Catalog>()
        try {
            val saxReader = SAXReader()
            val inputStream = context.resources.assets.open("contents.xml")
            val doc = saxReader.read(inputStream)
            val rootElement = doc.rootElement
            parseChild(1, rootElement, catalogs)
        } catch (e: java.lang.Exception) {
            e.printStackTrace()
        }

        return catalogs
    }

    private fun parseChild(level: Int, root: org.dom4j.Element, catalogs: ArrayList<Catalog>) {
        if (root == null) return
        val elements = root.elements()
        for (i in 0 until elements.size) {
            val element = elements.get(i)
            val childElement = element.elements()
            val catalog = Catalog()
            catalog.name = element.attributeValue("name")
            catalog.startPage = element.attributeValue("start-page")
            catalog.endPage = element.attributeValue("end-page")
            catalog.level = level
            catalogs.add(catalog)
            if (childElement.size > 0) {
                parseChild(level + 1, element, catalogs)
            }
        }
    }
```









