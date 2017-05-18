---
layout: post
title:  "Python 实战 爬取某漫画网站"
date:   2017-04-19 09:39:30 +0800
categories: python
---

* TOC
{:toc}

学习了 python 基础知识以后，想做个实际的项目练练手，以前常常看漫画，这次就做个漫画的爬虫好了。

## 分析过程

### 查看网页元素
在网页里导航 http://m.ikanman.com/comic/9637/97814.html 这个网址，用开发者工具看一下下载下来的图片位于哪个 html 标签里：

![]({{site.url}}/asset/python-comic-spider-img-node.png)

可以看到图片放在一个 img 标签里，这个标签不是一开始就存放在 html 文件里的，而是后期通过 js 动态设置上去的。因为直接下载网页的话并不会有这个标签。

### 查看哪里使用了图片

直接看 js 代码太累，不妨搜索一下这个图片的名字 000001.jpg，看哪段 js 代码使用了它，方便定位到代码位置：

![]( {{site.url}}/asset/python-comic-spider-img-js.png )

可以看到是在 VM388 这个地方包含了图片名，并且它包含了该漫画所有的下载路径(相对路径)。现在看来，如果能找到 VM388 的来源，就能获取漫画的下载地址列表了。

VM388 这个东西是 chrome 自动格式化出来的一个文件，并不存在对应的 js 文件，一般是类似于 `window.eval` 所执行的代码。所以直接搜索是找不到它的 js 文件的，一定是某个地方构造了它。

### 分析 VM388 的来源

VM388 里有一个对象 SMH.reader，搜索之后发现是 `core_75A66E6FE107617089CE900FA7A2876E.js` 这个文件里使用到了它：

![]( {{ site.url }}/asset/python-comic-spider-smh-reader.png )

可以看到 SMH.reader 是一个函数，函数接收两个参数，这两个参数也就是 VM388 里的那些代码。结合 VM388 的作用，可以推断是某个地方使用 `window.eval` 调用了 `SMH.reader`, 并将漫画信息作为参数传进去了。我所要找的漫画信息不是在 SMH.reader 这个函数里，而是在其他的某个地方，所以 SMH.reader 这个函数就不用看了。

在 SMH.reader 里设一个断点，按 F5 运行之后看是哪里调用了它：

![]( {{ site.url }}/asset/python-comic-spider-smh-reader-callstack.png )

在调用堆栈里一层层网上找，发现是在一开始的 html 文件里调用的。可惜 chrome 的开发者工具并没有帮我定位到 html 文件中代码的具体位置。

接下来就靠猜测了，首先 SMH.reader 函数是在 core_75A66E6FE107617089CE900FA7A2876E.js 文件里定义的，那么调用 SMH.reader 的代码一定是在 `<script type="text/javascript" src="http://c.3qfm.com/scripts/core_75A66E6FE107617089CE900FA7A2876E.js"></script>` 这个节点的后面。所幸 html 中的 `<script>` 节点不多，我发现了下面这个节点很可疑：

```html
<script type="text/javascript">
    window["\x65\x76\x61\x6c"](function(p, a, c, k, e, d) {
        e = function(c) { return (c < a ? "" : e(parseInt(c / a))) + ((c = c % a) > 35 ? String.fromCharCode(c + 29) : c.toString(36)) };
        if (!''.replace(/^/, String)) {
            while (c--) d[e(c)] = k[c] || e(c);
            k = [function(e) { return d[e] }];
            e = function() { return '\\w+' };
            c = 1;
        };
        while (c--)
            if (k[c]) p = p.replace(new RegExp('\\b' + e(c) + '\\b', 'g'), k[c]);
        return p;
    }('A.z({"B":D,"C":"u","t":v,"x":"w-K","J":L,"N":0,"M":["/7/y/2/4-3/F.6.5",\
    "/7/y/2/4-3/E.6.5","/7/y/2/4-3/G.6.5","/7/y/2/4-3/I.6.5","/7/y/2/4-3/H.6.5",\
    "/7/y/2/4-3/s.6.5","/7/y/2/4-3/g.6.5","/7/y/2/4-3/d.6.5","/7/y/2/4-3/c.6.5",\
    "/7/y/2/4-3/b.6.5","/7/y/2/4-3/e.6.5","/7/y/2/4-3/h.6.5","/7/y/2/4-3/f.6.5",\
    "/7/y/2/4-3/8.6.5","/7/y/2/4-3/9.6.5","/7/y/2/4-3/a.6.5","/7/y/2/4-3/o.6.5",\
    "/7/y/2/4-3/n.6.5","/7/y/2/4-3/p.6.5","/7/y/2/4-3/r.6.5","/7/y/2/4-3/q.6.5",\
    "/7/y/2/4-3/j.6.5","/7/y/2/4-3/i.6.5","/7/y/2/4-3/k.6.5","/7/y/2/4-3/m.6.5",\
    "/7/y/2/4-3/l.6.5","/7/y/2/4-3/1e.6.5","/7/y/2/4-3/1d.6.5","/7/y/2/4-3/1f.6.5",\
    "/7/y/2/4-3/1h.6.5","/7/y/2/4-3/1g.6.5","/7/y/2/4-3/19.6.5","/7/y/2/4-3/18.6.5",\
    "/7/y/2/4-3/1a.6.5","/7/y/2/4-3/1c.6.5","/7/y/2/4-3/1b.6.5","/7/y/2/4-3/1q.6.5",\
    "/7/y/2/4-3/1o.6.5","/7/y/2/4-3/1p.6.5","/7/y/2/4-3/1n.6.5","/7/y/2/4-3/1j.6.5",\
    "/7/y/2/4-3/1i.6.5","/7/y/2/4-3/1k.6.5","/7/y/2/4-3/1m.6.5","/7/y/2/4-3/1l.6.5",\
    "/7/y/2/4-3/17.6.5","/7/y/2/4-3/U.6.5","/7/y/2/4-3/T.6.5","/7/y/2/4-3/W.6.5",\
    "/7/y/2/4-3/V.6.5","/7/y/2/4-3/S.6.5","/7/y/2/4-3/P.6.5","/7/y/2/4-3/O.6.5",\
    "/7/y/2/4-3/R.6.5","/7/y/2/4-3/Q.6.5","/7/y/2/4-3/X.6.5","/7/y/2/4-3/14.6.5",\
    "/7/y/2/4-3/13.6.5","/7/y/2/4-3/16.6.5","/7/y/2/4-3/15.6.5"],"12":Z,"Y":1,"11":""}).10();', 62, 89,
    'D7CeEsEcFcEMDsDGALWB7ATgU3sATAMxgBGeADMAO5bEAOwAVrQObC0DOewZAjAbwBZuPAbwCsws\
    bwBs3MgE5eFMmQAcK+XIDsK1cJU8ew8nznSVW4T15cyecoW55r9p/zxC7UvLLui8Erw6PHoyQcLqPJ\
    p2Ks7CiuRyUmSyKLC0AC5YGACSACbAgABygM9GgKGxgF1ygPnKgDrygBJOwPJaqiLAgDTeiWmZ2QAq\
    4BkANlgg2LB52cAAygCyABLAxGhoANb584tLAHKwALZD8tIElgaxciq8cuRkRCqiZIFXN8DwWAAeGa\
    uEgLvRDU08EuDbWDMLDsNjYABuqzu5DE1zE1jEtjEojEgVhd08yXh3AEOgEoQE5lxOMUUhx6gE0VRd1\
    k7AysAy0FB5jBWBy8D6836aEQSwA+ohEMBEGhoPAMtwxDoxKExOYpZLFCy7uoxNEBFJCdwCOQCNcCNY\
    ddr+ARPAQpARfKaroE8Do8KEfHYjg67NF+FcjG6HjjrAJbLcBF7/WQBNcBKINTj+BHtYpRNqdARQsmr\
    u7zIcgA==' ['\x73\x70\x6c\x69\x63']('\x7c'), 0, {}))
</script>
```

### 分析 html 中可疑的 script 脚本

上面脚本中的 `window["\x65\x76\x61\x6c"]` 也就是 window.eval 的 ascii 码, 看起来似乎就是它调用了 SMH.reader, 把它放到开发者工具的 console 里执行一下，发现会报错：

![]( {{site.url}}/asset/python-comic-spider-eval-error.png )

上面的报错是我在其他页面(比如 www.google.com 页面下打开开发者工具) 的 console 里执行时才会报错的，在漫画网页本身的 console 里执行并不会报错。可以看到报错的原因是 split 不存在，也就是说，漫画网页中注册了一个 splic 这样的函数，但其他网页里没有注册，这样才能解释为什么漫画网页本身不会报错，其他页面却会报错。

很容易可以发现，splic 函数实际上就是上面的代码中的 `\x73\x70\x6c\x69\x63` 这段字符串。也就是说，script 脚本中有一部分代码可以翻译为：

```js
'D7CeEsEcFcEMDsDGALWB7ATgU3sATAMxgBGeADMAO5bEAOwAVrQObC0DOewZAjAbwBZuPAbwCsws\
bwBs3MgE5eFMmQAcK+XIDsK1cJU8ew8nznSVW4T15cyecoW55r9p/zxC7UvLLui8Erw6PHoyQcLqPJ\
p2Ks7CiuRyUmSyKLC0AC5YGACSACbAgABygM9GgKGxgF1ygPnKgDrygBJOwPJaqiLAgDTeiWmZ2QAq\
4BkANlgg2LB52cAAygCyABLAxGhoANb584tLAHKwALZD8tIElgaxciq8cuRkRCqiZIFXN8DwWAAeGa\
uEgLvRDU08EuDbWDMLDsNjYABuqzu5DE1zE1jEtjEojEgVhd08yXh3AEOgEoQE5lxOMUUhx6gE0VRd1\
k7AysAy0FB5jBWBy8D6836aEQSwA+ohEMBEGhoPAMtwxDoxKExOYpZLFCy7uoxNEBFJCdwCOQCNcCNY\
ddr+ARPAQpARfKaroE8Do8KEfHYjg67NF+FcjG6HjjrAJbLcBF7/WQBNcBKINTj+BHtYpRNqdARQsmr\
u7zIcgA==' ['splic']('|')
```

看起来上面这段代码前面一部分是一个 base64 字符串，用 base64 decode 工具解码一下，解出来是乱码。

既然字符串本身解析不出来，那就搜搜 splic 函数在哪里定义的吧，搜索一圈之后发现 splic 是在 VM797 这里定义的：

![]( {{site.url}}/asset/python-comic-spider-split.png)

虽然现在还不太懂 javascript 的 prototype, 但可以推断出来，上述这段代码把 splic 这个方法赋值给了字符串类，所以可以直接用字符串去调用 splic 函数。可以看到 splic 函数的定义是先调用 `LZString.decompressFromBase64` 对字符串做一个解压缩的处理，然后再调用字符串的 split 方法，把字符串分解成一个数组。

### 分析 LZString

`LZString.decopressFromBase64` 这个函数已经在 VM797 里定义好了，不过我还是比较好奇是什么地方构造了 VM797。

我猜是在 core_75A66E6FE107617089CE900FA7A2876E.js 里进行了这一步操作，所以大致浏览了一下代码，果不其然发现有一处 `window["\x65\x76\x61\x6c"]` 的代码，执行一下之后这段代码返回了一个函数，其内容就是 `LZString.decopressFromBase64`。

原始代码太长就不贴了，只贴一下执行结果的截图： 

![]( {{site.url}}/asset/python-comic-spider-eval-lzstring.png )

上面的代码应该是做了某种转码操作，把一堆 ascii 码翻译成了 LZString 的定义。

我尝试分析了上述代码的执行过程，不过还是没搞明白，后来转念一想，觉得自己钻了牛角尖，我需要知道的只是 LZString 这个类的功能和作用是什么，怎么把它转码出来其实没什么价值。

直接去看 VM797 里 LZString 的代码，也还是没搞明白这段代码在做什么。。。看函数名应该是压缩和解压，抱着试一试的心态去搜索了 LZString 关键字，发现它其实是一个用于压缩解压的 JavaScript 库：[LZString](http://pieroxy.net/blog/pages/lz-string/index.html)。

把这个库下载下来，带入刚刚的字符串运行一下，发现 `LZString.decopressFromBase64` 执行的结果是一个字符串，这个字符串随后会调用 split 方法分解为一个数组：

```js
LZString.decompressFromBase64('D7CeEsEcFcEMDsDGALWB7ATgU3sATAMxgBGeADMAO5bEAOwAVrQObC0DOewZAjAbwBZuPAbwCswsbwBs3MgE5eFMmQAcK+XIDsK1cJU8ew8nznSVW4T15cyecoW55r9p/zxC7UvLLui8Erw6PHoyQcLqPJp2Ks7CiuRyUmSyKLC0AC5YGACSACbAgABygM9GgKGxgF1ygPnKgDrygBJOwPJaqiLAgDTeiWmZ2QAq4BkANlgg2LB52cAAygCyABLAxGhoANb584tLAHKwALZD8tIElgaxciq8cuRkRCqiZIFXN8DwWAAeGauEgLvRDU08EuDbWDMLDsNjYABuqzu5DE1zE1jEtjEojEgVhd08yXh3AEOgEoQE5lxOMUUhx6gE0VRd1k7AysAy0FB5jBWBy8D6836aEQSwA+ohEMBEGhoPAMtwxDoxKExOYpZLFCy7uoxNEBFJCdwCOQCNcCNYddr+ARPAQpARfKaroE8Do8KEfHYjg67NF+FcjG6HjjrAJbLcBF7/WQBNcBKINTj+BHtYpRNqdARQsmru7zIcgA==');

// 以下是 decompressFromBase64 解码出来的字符串
"||yiquanchaoren|23|yb20|webp|jpg|ps2|013014|014015|015016|009010|008009|007008|010011|012013|006007|011012|022023|021022|023024|025026|024025|017018|016017|018019|020021|019020|005006|chapterId|一拳超人原作版|97814|第20|chapterTitle||reader|SMH|bookId|bookName|9637|001002|000001|002003|004005|003004|nextId|23话|97815|images|prevId|052053|051052|054055|053054|050051|047048|046047|049050|048049|055056|status|60|preInit|block_cc|count|057058|056057|059060|058059|045046|032033|031032|033034|035036|034035|027028|026027|028029|030031|029030|041042|040041|042043|044045|043044|039040|037038|038039|036037"

// 以下是对这个字符串调用 split 方法执行的结果
["", "", "yiquanchaoren", "23", "yb20", "webp", "jpg", "ps2", "013014", "014015", "015016", "009010", "008009", "007008", "010011", "012013", "006007", "011012", "022023", "021022", "023024", "025026", "024025", "017018", "016017", "018019", "020021", "019020", "005006", "chapterId", "一拳超人原作版", "97814", "第20", "chapterTitle", "", "reader", "SMH", "bookId", "bookName", "9637", "001002", "000001", "002003", "004005", "003004", "nextId", "23话", "97815", "images", "prevId", "052053", "051052", "054055", "053054", "050051", "047048", "046047", "049050", "048049", "055056", "status", "60", "preInit", "block_cc", "count", "057058", "056057", "059060", "058059", "045046", "032033", "031032", "033034", "035036", "034035", "027028", "026027", "028029", "030031", "029030", "041042", "040041", "042043", "044045", "043044", "039040", "037038", "038039", "036037"]
```

哇哦，经过 LZString 算法的处理，我们看到了那串 base64 字符的原始信息！其中包含了最值得关心的页码！不过现在它还是乱的，得看看它是怎么处理成 VM388 里面那样有顺序的结构的。

另外还有一个问题，现在的 LZString 库是 js 写的，我得想办法把它搞到 python 里，毕竟现在的目的是用漫画爬虫练习 python。虽然以前对 LZW 算法有所研究，但光看代码还是没看出个所以然，顺手搜索了一下 LZString python, 发现已经有人写了这样一个库：[lz-string-python](https://github.com/eduardtomasek/lz-string-python)。

用这个库尝试解析一下 base64 字符串，发现执行的时候会报错：

![]( {{ site.url }}/asset/python-comic-spider-lzstring-error.png)

老实说这个问题我没有彻底搞清楚原因，只是对比 js 和 python 的执行结果，发现 python 版本的 lzstring 库在执行到结尾的时候会解析错误，简单地加上一个防御措施，如果执行到结尾，就不继续运行，而是直接 return, 试了一下还真奏效，具体可以参考我 [github](https://github.com/Dong-dada/comic_spider) 中的代码进行对比。这儿就先不追究这个报错问题了，把代码分析完再说。

### 分析 html 可疑脚本中的另外一段代码

好了，回到之前的结论，html 中有一段脚本很可疑，现在我们已经知道了这段脚本的一个参数是用 LZString 库解码出来的字符串，随后经过 split 操作变成了一个数组，那么这段脚本就等价于如下代码：

```js
// 去掉了 window.eval
function foo(p, a, c, k, e, d) {
    e = function(c) { return (c < a ? "" : e(parseInt(c / a))) + ((c = c % a) > 35 ? String.fromCharCode(c + 29) : c.toString(36)) };
    if (!''.replace(/^/, String)) {
        while (c--) d[e(c)] = k[c] || e(c);
        k = [function(e) { return d[e] }];
        e = function() { return '\\w+' };
        c = 1;
    };
    while (c--)
        if (k[c]) p = p.replace(new RegExp('\\b' + e(c) + '\\b', 'g'), k[c]);
    return p;
};

foo('A.z({"B":D,"C":"u","t":v,"x":"w-K","J":L,"N":0,"M":["/7/y/2/4-3/F.6.5",\
"/7/y/2/4-3/E.6.5","/7/y/2/4-3/G.6.5","/7/y/2/4-3/I.6.5","/7/y/2/4-3/H.6.5",\
"/7/y/2/4-3/s.6.5","/7/y/2/4-3/g.6.5","/7/y/2/4-3/d.6.5","/7/y/2/4-3/c.6.5",\
"/7/y/2/4-3/b.6.5","/7/y/2/4-3/e.6.5","/7/y/2/4-3/h.6.5","/7/y/2/4-3/f.6.5",\
"/7/y/2/4-3/8.6.5","/7/y/2/4-3/9.6.5","/7/y/2/4-3/a.6.5","/7/y/2/4-3/o.6.5",\
"/7/y/2/4-3/n.6.5","/7/y/2/4-3/p.6.5","/7/y/2/4-3/r.6.5","/7/y/2/4-3/q.6.5",\
"/7/y/2/4-3/j.6.5","/7/y/2/4-3/i.6.5","/7/y/2/4-3/k.6.5","/7/y/2/4-3/m.6.5",\
"/7/y/2/4-3/l.6.5","/7/y/2/4-3/1e.6.5","/7/y/2/4-3/1d.6.5","/7/y/2/4-3/1f.6.5",\
"/7/y/2/4-3/1h.6.5","/7/y/2/4-3/1g.6.5","/7/y/2/4-3/19.6.5","/7/y/2/4-3/18.6.5",\
"/7/y/2/4-3/1a.6.5","/7/y/2/4-3/1c.6.5","/7/y/2/4-3/1b.6.5","/7/y/2/4-3/1q.6.5",\
"/7/y/2/4-3/1o.6.5","/7/y/2/4-3/1p.6.5","/7/y/2/4-3/1n.6.5","/7/y/2/4-3/1j.6.5",\
"/7/y/2/4-3/1i.6.5","/7/y/2/4-3/1k.6.5","/7/y/2/4-3/1m.6.5","/7/y/2/4-3/1l.6.5",\
"/7/y/2/4-3/17.6.5","/7/y/2/4-3/U.6.5","/7/y/2/4-3/T.6.5","/7/y/2/4-3/W.6.5",\
"/7/y/2/4-3/V.6.5","/7/y/2/4-3/S.6.5","/7/y/2/4-3/P.6.5","/7/y/2/4-3/O.6.5",\
"/7/y/2/4-3/R.6.5","/7/y/2/4-3/Q.6.5","/7/y/2/4-3/X.6.5","/7/y/2/4-3/14.6.5",\
"/7/y/2/4-3/13.6.5","/7/y/2/4-3/16.6.5","/7/y/2/4-3/15.6.5"],"12":Z,"Y":1,"11":""}).10();', 62, 89,
["", "", "yiquanchaoren", "23", "yb20", "webp", "jpg", "ps2", "013014", "014015", 
"015016", "009010", "008009", "007008", "010011", "012013", "006007", "011012", 
"022023", "021022", "023024", "025026", "024025", "017018", "016017", "018019", 
"020021", "019020", "005006", "chapterId", "一拳超人原作版", "97814", "第20", 
"chapterTitle", "", "reader", "SMH", "bookId", "bookName", "9637", "001002", 
"000001", "002003", "004005", "003004", "nextId", "23话", "97815", "images", 
"prevId", "052053", "051052", "054055", "053054", "050051", "047048", "046047", 
"049050", "048049", "055056", "status", "60", "preInit", "block_cc", "count",
 "057058", "056057", "059060", "058059", "045046", "032033", "031032", "033034",
  "035036", "034035", "027028", "026027", "028029", "030031", "029030", "041042",
   "040041", "042043", "044045", "043044", "039040", "037038", "038039", "036037"], 0, {})
```

上述代码的执行结果就是我在最开始的 VM388 里看到的那段字符串!

```js
"SMH.reader({"bookId":9637,"bookName":"一拳超人原作版","chapterId":97814,"chapterTitle":"第20-23话","nextId":97815,"prevId":0,"images":["/ps2/y/yiquanchaoren/yb20-23/000001.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/001002.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/002003.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/003004.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/004005.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/005006.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/006007.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/007008.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/008009.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/009010.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/010011.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/011012.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/012013.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/013014.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/014015.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/015016.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/016017.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/017018.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/018019.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/019020.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/020021.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/021022.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/022023.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/023024.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/024025.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/025026.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/026027.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/027028.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/028029.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/029030.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/030031.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/031032.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/032033.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/033034.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/034035.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/035036.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/036037.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/037038.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/038039.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/039040.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/040041.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/041042.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/042043.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/043044.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/044045.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/045046.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/046047.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/047048.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/048049.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/049050.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/050051.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/051052.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/052053.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/053054.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/054055.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/055056.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/056057.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/057058.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/058059.jpg.webp","/ps2/y/yiquanchaoren/yb20-23/059060.jpg.webp"],"count":60,"status":1,"block_cc":""}).preInit();"
```

ok，由此可以断定，这段代码的作用是将 LZString 解码后乱序的数组按照某种规则重新排序整合，改装为一个有顺序的结构。现在的问题是，这个转换函数的转换原理是什么？

### 分析 html 可疑脚本中的转换函数

现在的核心问题是搞懂这个转换函数的原理，然后把它移植到 python 里面。

```js
function foo(p, a, c, k, e, d) {
    e = function(c) { return (c < a ? "" : e(parseInt(c / a))) + ((c = c % a) > 35 ? String.fromCharCode(c + 29) : c.toString(36)) };
    if (!''.replace(/^/, String)) {
        while (c--) d[e(c)] = k[c] || e(c);
        k = [function(e) { return d[e] }];
        e = function() { return '\\w+' };
        c = 1;
    };

    // 这里加个日志，看看此时各个值是什么内容

    while (c--)
        if (k[c]) p = p.replace(new RegExp('\\b' + e(c) + '\\b', 'g'), k[c]);
    return p;
};
```

通过打日志加断点的方式，可以看到上述代码的前半段是通过我们传入的数组 k 构造了一个字典 d:

```js
{"0":"0","1":"1","2":"yiquanchaoren","3":"23","4":"yb20","5":"webp","6":"jpg",
"7":"ps2","8":"013014","9":"014015","10":"preInit","11":"block_cc","12":"count",
"13":"057058","14":"056057","15":"059060","16":"058059","17":"045046","18":"032033",
"19":"031032","1q":"036037","1p":"038039","1o":"037038","1n":"039040","1m":"043044",
"1l":"044045","1k":"042043","1j":"040041","1i":"041042","1h":"029030","1g":"030031",
"1f":"028029","1e":"026027","1d":"027028","1c":"034035","1b":"035036","1a":"033034",
"Z":"60","Y":"status","X":"055056","W":"048049","V":"049050","U":"046047","T":"047048",
"S":"050051","R":"053054","Q":"054055","P":"051052","O":"052053","N":"prevId",
"M":"images","L":"97815","K":"23话","J":"nextId","I":"003004","H":"004005","G":"002003",
"F":"000001","E":"001002","D":"9637","C":"bookName","B":"bookId","A":"SMH","z":"reader",
"y":"y","x":"chapterTitle","w":"第20","v":"97814","u":"一拳超人原作版","t":"chapterId",
"s":"005006","r":"019020","q":"020021","p":"018019","o":"016017","n":"017018",
"m":"024025","l":"025026","k":"023024","j":"021022","i":"022023","h":"011012",
"g":"006007","f":"012013","e":"010011","d":"007008","c":"008009","b":"009010","a":"015016"}
```

这个构造方法是怎么回事儿呢？一开始我尝试分析了它的构造过程，因为有递归和一系列计算的原因，分析的过程并不顺利，后来想到反正这个算法的作用是构造，那么能不能对比数组 k 和字典 d 直接分析出来构造原理呢？

把 k 里各个元素的索引和 d 里各个元素的 key 值对比一下，很容易发现了构造原理：数组的前 10 个元素分别映射到了字典中 `0~9` 的键，接下来 26 个元素映射到了字典中 `a~z` 的键，再接下来的 26 个元素映射到了字典中 `A~Z` 的 key, 再接下来的 10 个元素映射到了字典中 `10~19` 的键，剩下的几个元素映射到了字典中 `1a~1q` 的键。

也就是说，它的映射规律是：

```
// 上面一行是数组 k 的索引，下面一行是字典 d 的 key

0~9   10~35   36~61   62~71   72~97   98~123  124~133  134~159  160~185  186~195  …
 |      |       |       |       |       |        |        |        |        |
0~9    a~z     A~Z    10~19   1a~1z   1A~1Z    20~29    2a~2z    2A~2Z    30~39    …
```

搞明白了这件事，foo 里的逻辑就清楚了一大半，我把这个函数里加上注释解释一下：

```js
// 函数 e 的作用是将数组 k 的 index 映射为字典 d 的 key
e = function(c) { return (c < a ? "" : e(parseInt(c / a))) + ((c = c % a) > 35 ? String.fromCharCode(c + 29) : c.toString(36)) };

if (!''.replace(/^/, String)) {
    // 构造字典 d
    while (c--) d[e(c)] = k[c] || e(c);

    // 这行代码将 k 改成了一个数组，它只有一个函数，函数的作用是根据 key 返回字典 d 中对应的 value
    k = [function(e) { return d[e] }];

    // 这行代码将 e 改成了一个函数，它的作用是返回 `\\w+`
    e = function() { return '\\w+' };
    
    // 这里将 c 改成了 1
    c = 1;
};

// while 循环实际上没意义，因为 c = 1 所以只会执行一次
while (c--) {
    // if 判断实际上没意义，因为 c = 0 所以实际上 k[c] 必定不为空
    if (k[c]) {
        // 这行代码中的 e(c) 实际上就是 '\\w+' 而 k[c] 实际上是 k[0], 也就是那个函数
        p = p.replace(new RegExp('\\b' + e(c) + '\\b', 'g'), k[c]);
    }
}

return p;
```

可以看到，字典 d 构造成功后，实际上传给了一个正则表达式替换函数，等价于：

```js
p = p.replace(new RegExp('\\b\\w+\\b', 'g'), function(e){
    return d[e];
});
```

也就是说，找到字符串 p 里的每个单词，将其替换为 d 里对应的 value。字符串 p 也就是下面这串内容：

```js
'A.z({"B":D,"C":"u","t":v,"x":"w-K","J":L,"N":0,"M":["/7/y/2/4-3/F.6.5",\
"/7/y/2/4-3/E.6.5","/7/y/2/4-3/G.6.5","/7/y/2/4-3/I.6.5","/7/y/2/4-3/H.6.5",\
"/7/y/2/4-3/s.6.5","/7/y/2/4-3/g.6.5","/7/y/2/4-3/d.6.5","/7/y/2/4-3/c.6.5",\
"/7/y/2/4-3/b.6.5","/7/y/2/4-3/e.6.5","/7/y/2/4-3/h.6.5","/7/y/2/4-3/f.6.5",\
"/7/y/2/4-3/8.6.5","/7/y/2/4-3/9.6.5","/7/y/2/4-3/a.6.5","/7/y/2/4-3/o.6.5",\
"/7/y/2/4-3/n.6.5","/7/y/2/4-3/p.6.5","/7/y/2/4-3/r.6.5","/7/y/2/4-3/q.6.5",\
"/7/y/2/4-3/j.6.5","/7/y/2/4-3/i.6.5","/7/y/2/4-3/k.6.5","/7/y/2/4-3/m.6.5",\
"/7/y/2/4-3/l.6.5","/7/y/2/4-3/1e.6.5","/7/y/2/4-3/1d.6.5","/7/y/2/4-3/1f.6.5",\
"/7/y/2/4-3/1h.6.5","/7/y/2/4-3/1g.6.5","/7/y/2/4-3/19.6.5","/7/y/2/4-3/18.6.5",\
"/7/y/2/4-3/1a.6.5","/7/y/2/4-3/1c.6.5","/7/y/2/4-3/1b.6.5","/7/y/2/4-3/1q.6.5",\
"/7/y/2/4-3/1o.6.5","/7/y/2/4-3/1p.6.5","/7/y/2/4-3/1n.6.5","/7/y/2/4-3/1j.6.5",\
"/7/y/2/4-3/1i.6.5","/7/y/2/4-3/1k.6.5","/7/y/2/4-3/1m.6.5","/7/y/2/4-3/1l.6.5",\
"/7/y/2/4-3/17.6.5","/7/y/2/4-3/U.6.5","/7/y/2/4-3/T.6.5","/7/y/2/4-3/W.6.5",\
"/7/y/2/4-3/V.6.5","/7/y/2/4-3/S.6.5","/7/y/2/4-3/P.6.5","/7/y/2/4-3/O.6.5",\
"/7/y/2/4-3/R.6.5","/7/y/2/4-3/Q.6.5","/7/y/2/4-3/X.6.5","/7/y/2/4-3/14.6.5",\
"/7/y/2/4-3/13.6.5","/7/y/2/4-3/16.6.5","/7/y/2/4-3/15.6.5"],"12":Z,"Y":1,"11":""}).10();'
```

它的每个单词都会被替换为字典 d 里对应的那个值，比如单词 `A` 被替换为 `d["A"] = "SMH"`，如此一来就把我们一开始穿进去的数组构造成了最终的结构。

### 总结

现在我们了解到了漫画下载列表被解析的整个过程了：
- 网页首先执行 core_75A66E6FE107617089CE900FA7A2876E.js 代码，在这里声明了 LZString 及对应的 `splic` 方法；
- 网页执行内嵌的 js 代码，通过 LZString 和 split 方法将 Base64 编码的数据转换为数组；
- 网页执行内嵌的 js 代码，将数组通过映射关系表转换为字典；
- 网页执行内嵌的 js 代码，将字典中的元素通过正则表达式填充到字符串中；

这个过程中网页使用了如下保护措施：
- 网页内嵌的 js 代码用十六进制表示，加大阅读难度；
- 定义一个不认识的函数 splic, 把它隐藏在 core_75A66E6FE107617089CE900FA7A2876E.js 中；
- 通过 十六进制码 以及 转换函数 来保护 LZString 压缩算法；
- LZString 解压缩后获得的漫画信息作为一个数组，是乱序的，需要经过另外的处理才能拍好序；


## 编写爬虫代码

有了上述结论，可以整理出爬取网站的步骤：
- 下载网页；
- 得到内嵌的 js 代码，提取出其中最重要的两个参数：存放漫画详细信息的字符串，存放字段对应关系的字符串；
- 使用 LZString 算法对漫画详细信息字符串进行解码，得到漫画详细信息的数组；
- 使用之前分析出来的映射算法对数组进行处理，得到最终的字典对象；

其中 LZString 算法已经有了 python 版本，不需要自己做了，映射算法需要自己写一个，所幸不是很难。

以下是按照这个思路编写的爬虫代码，查看代码及参考注释可以很容易理解每一步的作用：

```py
#coding:utf-8

"""
comic spider
"""

import json
import os
import re
from urllib import request
from bs4 import BeautifulSoup
from lzstring import LZString

def main():
    """main"""

    url = 'http://m.ikanman.com/comic/9637/97814.html'
    html = None

    # download html
    req = request.Request(url)
    req.add_header('User-Agent', 'Mozilla/6.0 (iPhone; CPU iPhone OS 8_0 like Mac OS X) AppleWebKit/536.26 (KHTML, like Gecko) Version/8.0 Mobile/10A5376e Safari/8536.25')
    with request.urlopen(req) as response:
        if response.status != 200:
            print("download failed! status = %s, reason = %s" % (response.status, response.reason))
            return

        html = response.read()

    # get script node
    soup = BeautifulSoup(html, 'html.parser')
    node = soup.find('script', text=re.compile(r'^window'))
    if node is None:
        print("get script node failed!")
        return

    script = node.get_text()
    if script is None:
        print("get script text failed!")
        return

    # get dict_json and compress_str
    match_result = re.match(r'.*({".*"}).*\'([0-9a-zA-Z/\+=]+)\'.*', script)
    if match_result is None:
        print("re.match failed! script = %s" % script)
        return

    dict_json = match_result.group(1)
    compress_str = match_result.group(2)

    # decompress compress_str
    lzstring = LZString()
    array_str = lzstring.decompresFromBase64(compress_str)
    if array_str is None:
        print("lzstring.decompressFromBase64 failed! compress_str = %s" % compress_str)
        return

    # split array_str to array
    comic_info_arr = array_str.split('|')
    if comic_info_arr is None or len(comic_info_arr) == 0:
        print("split array_str failed! array_str = %s" % array_str)
        return

    # build comic_info_dict
    comic_info_dict = {}
    index_to_key_map = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"

    def index_to_key(index):
        """index_to_key"""

        key = ""

        if index == 0:
            return "0"

        while index > 0:
            key = index_to_key_map[index % 62] + key
            index = int(index / 62)
        return key

    for index in range(0, len(comic_info_arr)):
        key = index_to_key(index)
        if comic_info_arr[index] != "":
            comic_info_dict[key] = comic_info_arr[index]
        else:
            comic_info_dict[key] = key

    # replace dict_json
    def replace(match):
        """replace"""
        return comic_info_dict[match.group(0)]

    comic_info_json = re.sub(r'\b\w+\b', replace, dict_json)
    if comic_info_json is None:
        print("re.sub failed! dict_json = %s" % dict_json)
        return

    # parse json to comic_info
    comic_info = json.loads(comic_info_json)
    if comic_info is None:
        print("json.loads failed! comic_info_json = %s" % comic_info_json)
        return

    if comic_info["images"] is None or len(comic_info["images"]) == 0:
        print("comic_info.images is None or empty! comic_info.images = %s" % str(comic_info.images))
        return

    # add host
    comic_images = comic_info["images"]
    for i in range(0, len(comic_images)):
        comic_images[i] = "http://i.hamreus.com:8080" + comic_images[i]

    print("prepare download image count = %s" % len(comic_images))

    # download image
    image_dir = "./images"
    if not os.path.exists(image_dir):
        os.mkdir(image_dir)
    for image_url in comic_images:
        image_name = image_url.split('/')[-1]
        image_path = image_dir + "/" + image_name
        req = request.Request(image_url)
        req.add_header("Accept", "image/webp,image/*,*/*;q=0.8")
        req.add_header("Referer", "http://m.ikanman.com/comic/9637/97814.html")
        req.add_header("Connection", "keep-alive")
        req.add_header('User-Agent', 'Mozilla/5.0 (Windows NT 6.3; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/39.0.2171.95 Safari/537.36')
        image_data = request.urlopen(req).read()
        with open(image_path, "wb") as image_file:
            image_file.write(image_data)
            print("download success! image_name = %s, image_data size = %d" % (image_name, len(image_data)))

    print("main exit")


if __name__ == '__main__':
    main()
```

上面的代码只能爬取某一部漫画，并不是爬取全站所有漫画的，所以思路很简单。以后有兴趣可以写一个专门爬取漫画信息的爬虫，把全站漫画的信息做一个汇总，然后根据自己想法来下载任意一部漫画。