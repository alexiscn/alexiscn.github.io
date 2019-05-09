---
layout: post
title:  "开源项目V2EX"
date:   2018-12-12 22:33:10 +0800
categories: Swift iOS V2EX
tag: 开源项目
---

半年前给自己挖了一个坑，想做一个V2EX第三方的iOS客户端，在github上开了[repo](https://github.com/alexiscn/V2EX)，最近才抽空有时间把这个坑填上，目前完成度大致满足了我的日常需求，先PO靓照。

![](/assets/images/2018/v2ex_overview.png)

主要的功能有

* 两款皮肤，白天模式、黑夜模式
* 侧栏实现快速切换节点或者Tab
* 个性化个人资料面，可以查看他的主题，他创建的主题
* 账号登录，自动签到
* 多语言，目前支持简体中文、英文
* 评论、点赞
* 发帖子（TODO）

记录一下做这个项目的一些经验有意思的重构与架构，包括网络封装、MVVM列表、动态换肤。

## 封装网络请求

V2EX 官方其实也提供了[API](https://www.v2ex.com/p/7v9TEc53)，但API返回的结果数据太少，所以项目中使用的接口其实都是直接请求的HTML页面进行解析得到的。解析了几个网页后，发现网络请求有一些通用的地方：

* 查看网页可能需要登录，比如帖子设为登录用户可查看
* 网络错误的统一处理，如404、500
* 将HTML字符串转成 Document 对象
* 有一些的网页需要特殊的HTTP Headers，如登录需要设置`Referer`

于是就重构封装了下网络处理，包括 EndPoint、Response、HTMLParser 和 V2SDK 这四个部分。

### EndPoint

定义API的结构，包含`Path`、`HTTPMethod`、请求参数以及HTTPHeaders，同时需要实现`URLRequestConvertible`以便用`Alamofire`请求。由于暂时不涉及到上传bodyData到服务器，就没有封装post data相关的逻辑。

```swift
import Alamofire

struct EndPoint {
    
    let path: String
    
    let method: HTTPMethod
    
    let parameters: Parameters?
    
    let headers: HTTPHeaders?
    
    init(path: String, method: HTTPMethod = .get, parameters: Parameters? = nil, headers: HTTPHeaders? = nil) {
        self.path = path
        self.method = method
        self.parameters = parameters
        self.headers = headers
    }
}

extension EndPoint: URLRequestConvertible {
    
    internal var url: URL {
        return URL(string: V2SDK.baseURLString.appending(path))!
    }
    
    func asURLRequest() throws -> URLRequest {
        var request = URLRequest(url: url)
        request.httpMethod = method.rawValue
        var urlRequest = try URLEncoding().encode(request, with: parameters)
        urlRequest.allHTTPHeaderFields = headers
        return urlRequest
    }
}
```

这样每个接口就是一个EndPoint对象，我们可以在EndPoint的扩展里面写静态方法定义所有API接口，如下定义了V2EX主帧的API，参数为tab的key，以及需要设置特殊HTTP Header的登录接口。

```swift
extension EndPoint {
    
    static func tab(_ tab: String) -> EndPoint {
        let path = "/?tab=\(tab)"
        return EndPoint(path: path)
    }

    static func signIn(username: String, password: String, captcha: String, formData: LoginFormData) -> EndPoint {
        let path = "/signin"
        var params: [String: String] = [:]
        params["next"] = "/"
        params["once"] = formData.once
        params[formData.username] = username
        params[formData.password] = password
        params[formData.captcha] = captcha
        
        var headers = Alamofire.SessionManager.defaultHTTPHeaders
        headers["Referer"] = V2SDK.baseURLString + path
        headers["User-Agent"] = UserAgents.phone
        
        return EndPoint(path: path, method: .post, parameters: params, headers: headers)
    }
}
```

### Response

接下来就是定义请求返回数据结构，参考Alamofire的返回数据结构定义，返回结构是一个枚举。`success(T)`即成功回调，并且`T`不是`optional`类型的，这样在上层业务中不需要判断是否为nil。`error(V2Error)`即错误回调，V2Error定义了有哪些类型的错误。

```swift
enum V2Response<T> {
    case success(T)
    case error(V2Error)
}

enum V2Error: Error {
    case needsSignIn
    case needsTwoFactor
    case parseHTMLError
    case signInFailed
    case severNotFound
}
```

### HTMLParser

HTMLParser 是指定用来解析网页的处理类，实际上就是一个泛型协议，每个业务处理解析HTML逻辑，返回处理结果，解析HTML用的[SwiftSoup](https://github.com/scinfu/SwiftSoup)。

```swift
protocol HTMLParser {
    static func handle<T>(_ doc: Document) throws -> T?
}
```

以登录获取验证码来举例，实现`HTMLParser`协议，解析网页内容，返回处理结果。如果解析不到或者出现异常，直接抛出。在V2SDK层面会进行捕获异常，统一处理。

```swift
/// 登录验证码HTML解析
struct OnceTokenParser: HTMLParser {
    
    static func handle<T>(_ doc: Document) throws -> T? {
        let keys = try doc.select("input.sl").array()
        if keys.count == 3 {
            let username = try keys[0].attr("name")
            let password = try keys[1].attr("name")
            let captcha = try keys[2].attr("name")
            let once = try doc.select("input[name=once]").attr("value")
            let form = LoginFormData(username: username, password: password, captcha: captcha, once: once)
            return form as? T
        }
        throw V2Error.parseHTMLError
    }
}
```

如果需要给HTMLParser传一些参数的话，可以在实现类里面定义`static var` 的成员变量，在网络请求之前给其赋值即可，解析的时候直接使用成员变量的值。

```swift
struct UserTopicsParser: HTMLParser {
    
    static var avatarURL: URL?
    
    static func handle<T>(_ doc: Document) throws -> T? {
        var response: ListResponse<Topic> = ListResponse()
        if let max = try doc.select("input.page_input").first()?.attr("max") {
            response.page = Int(max) ?? 1
        }
        
        let cells = try doc.select("div.cell")
        for cell in cells {
            if !cell.hasClass("cell item") {
                continue
            }
            let topic = NodeTopicsParser.parseTopicListCell(cell)
            topic.avatar = avatarURL
            response.list.append(topic)
        }
        return response as? T
    }
}
```

### V2SDK

V2SDK是业务层直接使用的对象，他主要提供了一个请求接口，定义如下：

```swift
typealias RequestCompletionHandler<T> = (V2Response<T>) -> Void

class V2SDK {

    static let baseURLString = "https://www.v2ex.com"

    @discardableResult
    class func request<T>(_ endPoint: EndPoint, parser: HTMLParser.Type, completion: @escaping RequestCompletionHandler<T>) -> DataRequest {
        let dataRequest = Alamofire.request(endPoint)
        dataRequest.responseString { response in
            guard let html = response.value else {
                completion(V2Response.error(.severNotFound))
                return
            }
            // 需要登录才能访问
            if response.response?.url?.path == "/signin" && response.request?.url?.path != "/signin" {
                completion(V2Response.error(.needsSignIn))
                return
            }
            
            do {
                let doc = try SwiftSoup.parse(html)
                do {
                    let result: T? = try parser.handle(doc)
                    if let result = result {
                        completion(V2Response.success(result))
                    } else {
                        completion(V2Response.error(.serverNotFound))
                    }
                } catch let err as V2Error {
                    completion(V2Response.error(err))
                }
            } catch {
                print(error)
                completion(V2Response.error(.parseHTMLError))
            }
        }
        return dataRequest
    }
}
```

其中EndPoint 就是之前提到的API的信息封装，parser是将HTML解析成我们想要的返回数据，completion 是异步请求回调，回调参数是`V2Response<T>`泛型的结果。`request`方法体比较简单就不介绍了。

业务调用API就是请求V2SDK的request类方法。在请求回调里面处理相应的逻辑。比如请求主题明细页面的代码就可以简化为：

```swift
let endPoint = EndPoint.topicDetail(topicID)
V2SDK.request(endPoint, parser: TopicDetailParser.self) { [weak self] (response: V2Response<TopicDetail>) in
    guard let strongSelf = self else { return }
    // .... code to reset refreshing
    switch response {
    case .success(let detail):
        strongSelf.detail = detail
        // ... code to handle detail
    case .error(let error):
        HUD.show(message: error.description)
    }
}
```

如果想要取消网络请求，可以设置request的返回结果为成员变量，调用`cancel()`方法就取消网络请求了。


## 泛型MVVM列表页面

V2EX中有一些页面都是简单的分页列表，如用户创建的主题，用户所有回复的主题，收到的通知等。如果用传统的MVC写的话，有很多重复的代码，比如TableView的创建、数据的加载、页面的刷新等。稍微对这些列表页面进行抽象了下，他们满足几个条件

* 列表页面，可以下拉刷新、上滑加载更多
* 数据结构一致，即有页面总数，与页面数据列表
* 列表只有单一的CELL

所以抽象出来用MVVM模式来实现

### Model

定义Model比较简单，就是一个空的协议，业务层具体的model实现这个协议即可。

```swift
protocol DataType { }
```

### ViewModel

ViewModel 中主要把ViewController中的一些可变的提炼出来。如页面标题，数据源类型，注册的TableViewCell类型，请求的API，以及计算CELL的高度等等。

```swift

protocol ListViewModel: class {
    
    // model 类型
    associatedtype T: DataType
    
    var title: String? { get }
    
    // tableView的数据源
    var dataSouce: [T] { get set }
    
    // tableView 注册的UITableViewCell的类型
    var cellClass: UITableViewCell.Type { get }
    
    // 当前页码
    var currentPage: Int { get set }
    
    var endPoint: EndPoint { get }
    
    var apiParser: HTMLParser.Type { get }
    
    func heightForRowAt(_ indexPath: IndexPath) -> CGFloat
    
    func didSelectRowAt(_ indexPath: IndexPath)
}

```

### View

`View`主要是ViewController的逻辑，ListViewController的结构，是一个泛型的ViewController，类型即ViewModel的类型。


```swift
class ListViewController<T: ListViewModel>: UIViewController, UITableViewDelegate, UITableViewDataSource {
    
    private var tableView: UITableView!

    private var viewModel: T

    init(viewModel: T) {
        self.viewModel = viewModel
        super.init(nibName: nil, bundle: nil)
    }
    
    required init?(coder aDecoder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }

    override func viewDidLoad() {
        super.viewDidLoad()
    }

    fileprivate func setupTableView() {
        // setup tableView
        tableView = UITableView(frame: view.bounds)
        // ...
        tableView.register(viewModel.cellClass, forCellReuseIdentifier: NSStringFromClass(viewModel.cellClass))
        view.addSubview(tableView)
    }

    func numberOfSections(in tableView: UITableView) -> Int {
        return 1
    }
    
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return viewModel.dataSouce.count
    }
    
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let identifier = NSStringFromClass(viewModel.cellClass)
        let cell = tableView.dequeueReusableCell(withIdentifier: identifier, for: indexPath)
        let model = viewModel.dataSouce[indexPath.row]
        if let listCell = cell as? ListViewCell {
            listCell.update(model)
        }
        return cell
    }
    
    func tableView(_ tableView: UITableView, heightForRowAt indexPath: IndexPath) -> CGFloat {
        return viewModel.heightForRowAt(indexPath)
    }
    
    func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        tableView.deselectRow(at: indexPath, animated: false)
        viewModel.didSelectRowAt(indexPath)
    }
}
```


再来说一下网络请求，页面的网络请求结果结构都一样，一个列表数据，一个页码。可以用泛型定义如下：

```swift
struct ListResponse<T: DataType> {
    var list: [T] = []
    var page: Int = 1
}
```

```swift
extension ListViewModel {
    
    func loadData(isLoadMore: Bool, completion: @escaping ((ListDataInfo) -> Void)) {
        currentPage = isLoadMore ? (currentPage + 1): 1
        
        V2SDK.request(endPoint, parser: apiParser) { [weak self] (response: V2Response<ListResponse<T>>) in
            guard let strongSelf = self else { return }
            var info = ListDataInfo(isLoadMore: isLoadMore, canLoadMore: true)
            switch response {
            case .success(let result):
                if isLoadMore {
                    strongSelf.dataSouce.append(contentsOf: result.list)
                } else {
                    strongSelf.dataSouce = result.list
                }
                info.canLoadMore = strongSelf.currentPage < result.page
            case .error(let error):
                HUD.show(message: error.description)
            }
            completion(info)
        }
    }
}
```

消息通知页面，我们就只需要定义NotificationsViewModel、对应的HTML解析、以及编写CELL即可。

```swift
class NotificationsViewModel: ListViewModel {
    
    typealias DataType = MessageNotification
    
    var dataSouce: [MessageNotification] = []
    
    var cellClass: UITableViewCell.Type { return NotificationViewCell.self }
    
    var currentPage: Int = 1
    
    var endPoint: EndPoint { return EndPoint.notifications(page: currentPage) }
    
    var HTMLParser: HTMLParser.Type { return NotificationParser.self }
}
```

跳转到页面也十分简单，直接push即可。

```swift
let viewModel = BalanceViewModel()
let controller = ListViewController(viewModel: viewModel)
navigationController?.pushViewController(controller, animated: true)
```

### 往ViewModel中传参数

假如页面的加载需要外部的参数，可以在ViewModel的初始化函数中传入。以用户创建的主题列表为例，页面的地址为 `/member/{name}/topics`，name为该用户的用户名。

```swift
class UserTopicViewModel: ListViewModel {
    // ....

    var endPoint: EndPoint { return EndPoint.memberTopics(username) }

    let username: String
    
    init(username: String) {
        self.username = username
    }
}
```

### ViewModel 中进行页面跳转

ListViewModel中定义了选中TableViewCell的事件，在一些情况下需要进行页面的跳转，如跳转到主题详情页面。我们需要把 ViewController的`navigationController`传递到ViewModel中，以便在ViewModel中进行跳转。这里直接扩展`ListViewModel`协议。

```swift
protocol ListViewModel: class {
    
    // ....

    func didSelectRowAt(_ indexPath: IndexPath, navigationController: UINavigationController?)
}

class MyFavoritedTopicsViewModel: ListViewModel {

    // ...

    func didSelectRowAt(_ indexPath: IndexPath, navigationController: UINavigationController?) {
        let topic = dataSouce[indexPath.row]
        let detailVC = TopicDetailViewController(url: topic.url, title: topic.title)
        navigationController?.pushViewController(detailVC, animated: true)
    }
}

```

由于这个MVVM是针对V2EX定制的，所以就暂且实现这么多。


## 动态换肤

平时刷V站可能是白天，也可能是夜晚，所以就增加了两套皮肤。

### Theme

定义一个枚举，里面包含了Theme的类型：`light`、`dark`。以及还有当前的主题，注意是静态变量，因为可以动态修改当前的主题。

```swift
enum Theme: Int {
    case light
    case dark

        static var current: Theme = .dark
    
    var statusBarStyle: UIStatusBarStyle {
        switch self {
        case .light:
            return .default
        case .dark:
            return .lightContent
        }
    }
    
    var activityIndicatorViewStyle: UIActivityIndicatorView.Style {
        switch self {
        case .light:
            return .gray
        case .dark:
            return .white
        
        }
    }
    
    var titleColor: UIColor {
        switch self {
        case .light:
            return .black
        case .dark:
            return UIColor(red: 185.0/255, green: 200.0/255, blue: 243.0/255, alpha: 1.0)
        }
    }
}
```

在使用的时候，直接用当前皮肤的颜色即可。

```swift
titleLabel.textColor = Theme.current.titleColor
```

### 切换主题

[V2EX](https://github.com/alexiscn/V2EX) 里面使用了一个提交讨巧的设计，在侧栏中有一个按钮，点击后即切换主题。这样我们在切换主题的时候，只需要改首页的三个页面即可。

```swift 
class ThemeManager {

    static let shared = ThemeManager()

    private init() {}

    func observeThemeUpdated(closure: @escaping (Notification) -> Void) {
        let center = NotificationCenter.default
        center.addObserver(forName: NSNotification.Name.V2.ThemeUpdated, object: nil, queue: OperationQueue.main) { (notification) in
            closure(notification)
        }
    }
}
```

### WKWebView加载主题样式

在主题明细页面中使用`WKWebView`来展示主题内容，

```css
body {
    background-color:#2B3953;
    color: #B9C8F3;
}

a:link, a:visited, a:active {
    color:#147efb;
}
```



上面的代码都能在 [github v2ex](https://github.com/alexiscn/V2EX)上找到。