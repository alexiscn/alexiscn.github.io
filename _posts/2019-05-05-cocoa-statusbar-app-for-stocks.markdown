---
layout: post
title:  "Cocoa项目 - StocksBar"
date:   2019-05-05 11:33:10 +0800
tag: 开源项目
---

## 缘起

最近A股趋势不错，错过了之前的底部坑。于是就想着做一个Mac状态栏应用，可以在工作的时候偶尔关注一下股票的行情。

<img src='/assets/images/2019/stocksbar-home.png' width='300' />

主要的功能有

* 显示股票的当前价格、涨跌幅
* 搜索联想股票，添加到自选列表
* 拖拽编辑自选列表顺序
* 删除自选股股票
* 一些常用的配置选项，如轮询显示自选股、设置刷新间隔等

基本满足了我的日常需求。本文只要讲讲如何实现这个软件。

## 股票API

股票API用的新浪股票的接口：[http://hq.sinajs.cn/list=sh601933](http://hq.sinajs.cn/list=sh601933) 返回结果如下：

```js
var hq_str_sh601933="永辉超市,9.390,9.480,9.400,9.650,9.310,9.400,9.420,45031643,427349791.000,38767,9.400,19100,9.390,26500,9.380,14000,9.370,20400,9.360,1200,9.420,9300,9.430,39200,9.440,122072,9.450,26300,9.460,2019-05-09,15:00:00,00";
```

可以一次请求多个股票的行情，参数用逗号分隔即可。[https://hq.sinajs.cn/list=sh601933,sz000651](https://hq.sinajs.cn/list=sh601933,sz000651) 对应的返回值如下：

```js
var hq_str_sh601933="永辉超市,9.390,9.480,9.400,9.650,9.310,9.400,9.420,45031643,427349791.000,38767,9.400,19100,9.390,26500,9.380,14000,9.370,20400,9.360,1200,9.420,9300,9.430,39200,9.440,122072,9.450,26300,9.460,2019-05-09,15:00:00,00";
var hq_str_sz000651="格力电器,51.880,52.330,51.800,52.520,50.520,51.790,51.800,73733501,3800419825.700,5300,51.790,62400,51.780,6000,51.770,8700,51.760,5946,51.750,113623,51.800,17400,51.810,6400,51.820,4000,51.830,2200,51.840,2019-05-09,15:00:03,00";
```
| 字段索引 | 字段名称 | 数据实例 | 字段说明 |
| -- | -- | -- | -- |
| 0 | 股票简称 | 永辉超市 | |
| 1 | 今日开盘价 | 9.88 | |
| 2 | 昨日收盘价 | 9.66 | |
| 3 | 最近成交价 | 10.24 | |
| 4 | 最高成交价 | 10.88 | 当天最高成交价格 |
| 5 | 最低成交价 | 9.11 | |
| 6 | 买入价 | 10.25 | |
| 7 | 卖出价 | 10.31 | |
| 8 | 成交数量 | 5200 | 股为单位 |
| 9 | 成交金额 | xxx | |

我们对其中的股票名称、开盘价、当前价格、昨天收盘价等字段感兴趣，解析字符串的代码比较简单就不贴了。

定义对应的Model为

```swift
class Stock: NSObject, Codable {
    
    var code: String

    /// 股票简称
    var symbol: String = "永辉超市"
    
    /// 今日开盘价
    var openPrice: Float = 0.0
    
    /// 昨日收盘价
    var lastClosedPrice: Float = 0.0
    
    /// 最近成交价格
    var current: Float = 10.24
    
    /// 最高成交价
    var high: Float = 0.0
    
    /// 最低成交价
    var low: Float = 0.0

    init(code: String) {
        self.code = code
    }
}
```

### 搜索建议

新浪股票同样有一个搜索建议的api接口。[https://suggest3.sinajs.cn/suggest/type=&key=](https://suggest3.sinajs.cn/suggest/type=&key=)，参数key就是关键字，如果是中文需要urlencode。

比如我们搜索格力电器，输入geli，返回结果如下：

```js
var suggestvalue="格力电器,11,000651,sz000651,格力电器,,格力电器,99;格林美,11,002340,sz002340,格林美,,格林美,99;合力泰,11,002217,sz002217,合力泰,,合力泰,99;安徽合力,11,600761,sh600761,安徽合力,,安徽合力,99;18合力01,81,152035,sh152035,18合力01,,18合力01,99;18格力02,81,143870,sh143870,18格力02,,18格力02,99;18格力01,81,143869,sh143869,18格力01,,18格力01,99;PR合力01,81,124769,sh124769,PR合力01,,PR合力01,99;16合力01,81,112487,sz112487,16合力01,,16合力01,99;PR合力02,81,124770,sh124770,PR合力02,,PR合力02,99;格林国际控股,31,02700,02700,格林国际控股,,格林国际控股,99;香格里拉,31,00069,00069,香格里拉,,香格里拉,99;歌礼制药 B,31,01672,01672,歌礼制药 B,,歌礼制药 B,99;格菱控股,31,01318,01318,格菱控股,,格菱控股,99;格林货币B,24,004866,of004866,格林货币B,,格林货币B,99;博时合利货币,24,002960,of002960,博时合利货币,,博时合利货币,99;歌力思,11,603808,sh603808,歌力思,,歌力思,99;格林货币A,24,004865,of004865,格林货币A,,格林货币A,99;格力地产,11,600185,sh600185,格力地产,,格力地产,99;合力科技,11,603917,sh603917,合力科技,,合力科技,99;蒙古图格里克BRX,71,mntbrx,mntbrx,蒙古图格里克BRX,,蒙古图格里克BRX,99;乌克兰格里夫纳立陶宛立特参考汇率,71,uahltx,uahltx,乌克兰格里夫纳立陶宛立特参考汇率,,乌克兰格里夫纳立陶宛立特参考汇率,99;欧元蒙古图格里克,71,eurmnt,eurmnt,欧元蒙古图格里克,,欧元蒙古图格里克,99;美元蒙古图格里克,71,usdmnt,usdmnt,美元蒙古图格里克,,美元蒙古图格里克,99;瑞士法郎蒙古图格里克,71,chfmnt,chfmnt,瑞士法郎蒙古图格里克,,瑞士法郎蒙古图格里克,99;加拿大元乌克兰格里夫纳,71,caduah,caduah,加拿大元乌克兰格里夫纳,,加拿大元乌克兰格里夫纳,99;日元蒙古图格里克,71,jpymnt,jpymnt,日元蒙古图格里克,,日元蒙古图格里克,99;英镑蒙古图格里克,71,gbpmnt,gbpmnt,英镑蒙古图格里克,,英镑蒙古图格里克,99;加拿大元蒙古图格里克,71,cadmnt,cadmnt,加拿大元蒙古图格里克,,加拿大元蒙古图格里克,99;乌克兰格里夫纳RUX,71,uahrux,uahrux,乌克兰格里夫纳RUX,,乌克兰格里夫纳RUX,99;英镑乌克兰格里夫纳,71,gbpuah,gbpuah,英镑乌克兰格里夫纳,,英镑乌克兰格里夫纳,99;美元乌克兰格里夫纳,71,usduah,usduah,美元乌克兰格里夫纳,,美元乌克兰格里夫纳,99;乌克兰格里夫纳匈牙利福林参考汇率,71,uahhux,uahhux,乌克兰格里夫纳匈牙利福林参考汇率,,乌克兰格里夫纳匈牙利福林参考汇率,99;乌克兰格里夫纳波兰兹罗提参考汇率,71,uahplx,uahplx,乌克兰格里夫纳波兰兹罗提参考汇率,,乌克兰格里夫纳波兰兹罗提参考汇率,99;欧元乌克兰格里夫纳,71,euruah,euruah,欧元乌克兰格里夫纳,,欧元乌克兰格里夫纳,99;乌克兰格里夫纳BRX,71,uahbrx,uahbrx,乌克兰格里夫纳BRX,,乌克兰格里夫纳BRX,99;瑞士法郎乌克兰格里夫纳,71,chfuah,chfuah,瑞士法郎乌克兰格里夫纳,,瑞士法郎乌克兰格里夫纳,99;日元乌克兰格里夫纳,71,jpyuah,jpyuah,日元乌克兰格里夫纳,,日元乌克兰格里夫纳,99;乌克兰格里夫纳英镑,71,uahgbp,uahgbp,乌克兰格里夫纳英镑,,乌克兰格里夫纳英镑,99;招商盛合灵活混合C,21,004143,of004143,招商盛合灵活混合C,,招商盛合灵活混合C,99;格林伯盛混合C,21,004817,of004817,格林伯盛混合C,,格林伯盛混合C,99;银华合利债券,21,002306,of002306,银华合利债券,,银华合利债券,99;招商盛合灵活混合A,21,004142,of004142,招商盛合灵活混合A,,招商盛合灵活混合A,99;格林泓鑫纯债债券C,21,006185,of006185,格林泓鑫纯债债券C,,格林泓鑫纯债债券C,99;长信合利混合C,21,005306,of005306,长信合利混合C,,长信合利混合C,99;格林泓鑫纯债债券A,21,006184,of006184,格林泓鑫纯债债券A,,格林泓鑫纯债债券A,99;格林伯锐灵活配置混合A,21,006181,of006181,格林伯锐灵活配置混合A,,格林伯锐灵活配置混合A,99;格林伯元灵活配置混合A,21,004942,of004942,格林伯元灵活配置混合A,,格林伯元灵活配置混合A,99;格林伯锐灵活配置混合C,21,006182,of006182,格林伯锐灵活配置混合C,,格林伯锐灵活配置混合C,99;格林伯盛混合A,21,004816,of004816,格林伯盛混合A,,格林伯盛混合A,99";
```

可以看到返回的结构跟行情很类似，处理搜索的时候有一些细节要处理，比如在输入第二个字符的时候，应该把输入第一个字符的请求取消掉（如果结果还没有返回的话），以及停留多少秒就立马去处理请求的情况，这个在后续会讲到。


## NSTableView

`NSTableView` 与 `UITableView` 有相似的地方，也有很多不同的地方。


#### 滑动

NSTableView 本身不支持滑动，需要嵌套在 NSScrollView 中才能滑动。通过是直接在Storyboard或者Xib中进行布局，然后添加IBOutlet到ViewController中，也可以在代码中创建NSScrollView与NSTableView，设置 TableView为ScrollView的documentView。

```swift
scrollView = NSScrollView(frame: .zero)
scrollView.automaticallyAdjustsContentInsets = false
scrollView.drawsBackground = false
scrollView.hasVerticalScroller = true
scrollView.borderType = .noBorder
scrollView.backgroundColor = .clear
view.addSubview(scrollView)
scrollView.snp.makeConstraints { make in
    make.leading.trailing.equalToSuperview()
    make.top.equalToSuperview()
    make.bottom.equalToSuperview()
}
```

上传代码创建了

```swift
tableView = NSTableView()
tableView.rowHeight = 40.0
tableView.backgroundColor = NSColor(white: 1, alpha: 0.6)
tableView.register(NSNib(nibNamed: "StockTableCellView", bundle: nil), forIdentifier: reuseIdentifier)
tableView.selectionHighlightStyle = .none
tableView.dataSource = self
tableView.delegate = self
tableView.floatsGroupRows = true
tableView.intercellSpacing = NSSize.zero
tableView.registerForDraggedTypes([dragType])
tableView.draggingDestinationFeedbackStyle = .gap
scrollView.documentView = tableView

let column = NSTableColumn()
column.width = view.bounds.width
tableView.headerView = nil
tableView.addTableColumn(column)
```


#### 分隔符

通过设置tableView的 GridMaskStyle 可以设置分隔符的样式。但是不同于 UITableView有一个 tableFooterView的属性可以设置为空的视图可以将没有

### 纯代码 NSViewController

默认NSViewController 不能由代码实例化，如果 let controller = YourNSViewController 会报如下的错误信息

```swift
-[NSNib _initWithNibNamed:bundle:options:] could not load the nibName: ProjectName.YourNSViewController in bundle (null).
```

借助如下的代码

```swift
override func loadView() {
    self.view = NSView()
}
```

### 修改视图的背景颜色

不同于 UIView， NSView 并不提供 backgroundColor属性来修改自身的背景颜色

```swift
view.wantsLayer = true
view.layer.backgroundColor = NSColor.white.cgColor
```