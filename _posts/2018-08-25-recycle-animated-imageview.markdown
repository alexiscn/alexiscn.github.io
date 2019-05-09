---
layout: post
title:  "类似小红书欢迎页面的背景循环滚动"
date:   2018-08-25 21:12:10 +0800
categories: Swift iOS Animation
---

最近实现了类似小红书欢迎页面的循环滚动背景的效果，思路有两个：

### 1、使用两个同样大小的ImageView进行轮播

```swift
private var firstImageView: UIImageView 
private var secondImageView: UIImageView
```

1.1 第一个ImageView正常靠顶部布局，第二个ImageView的origin.y 是第一个ImageView的底部

```
public override func layoutSubviews() {
    super.layoutSubviews()
    
    let h = firstImageView.bounds.height
    let w = bounds.width
    firstImageView.frame = CGRect(x: 0, y: 0, width: w, height: h)
    secondImageView.frame = CGRect(x: 0, y: h, width: w, height: h)
}
```

1.2 然后对这两个ImageView同时做动画，将第一个ImageView的 origin.y = -height，第二个ImageView的 origin.y = 0

```swift
UIView.animate(withDuration: self.duration, delay: 0, options: [.curveLinear], animations: { [weak self] in
    self?.firstImageView.frame.origin.y = -pageHeight
    self?.secondImageView.frame.origin.y = 0
}) { _ in
    ...
}
```

1.3 在动画结束的时候，将第一个ImageView的 origin.y 设置为 height, 即把第一个ImageView放到第二个ImageView的下方，将第二个ImageView的 origin.y 设置为 0，同时交换两个ImageView

```swift
UIView.animate(withDuration: self.duration, delay: 0, options: [.curveLinear], animations: { [weak self] in
    ...
}) { [weak self] _ in
    guard let strongSelf = self else {
        return
    }
    strongSelf.firstImageView.frame.origin.y = pageHeight
    strongSelf.secondImageView.frame.origin.y = 0
    (strongSelf.firstImageView, strongSelf.secondImageView) = (strongSelf.secondImageView, strongSelf.firstImageView)
}
```

1.4 重复 2 - 4 即能做到无限循环播放。

```swift
UIView.animate(withDuration: self.duration, delay: 0, options: [.curveLinear], animations: { [weak self] in
    ...
}) { [weak self] _ in
    guard let strongSelf = self else {
        return
    }
    ...
    strongSelf.startAnimation()
}
```


### 2、使用UICollectionView

2.1 对 `contentOffset.y`进行动画。同时需要将UICollectionView的手势禁用


完整的代码：

```swift
@IBDesignable
public class RecycleAnimatedView: UIView {

    private var firstImageView: UIImageView
    
    private var secondImageView: UIImageView
    
    override public init(frame: CGRect) {
        firstImageView = UIImageView()
        secondImageView = UIImageView()
        super.init(frame: frame)
        commonInit()
    }
    
    required public init?(coder aDecoder: NSCoder) {
        firstImageView = UIImageView()
        secondImageView = UIImageView()
        super.init(coder: aDecoder)
        commonInit()
    }
    
    private func commonInit() {
        addSubview(firstImageView)
        addSubview(secondImageView)
        setNeedsDisplay()
    }
    
    public override func layoutSubviews() {
        super.layoutSubviews()
        
        let h = firstImageView.bounds.height
        let w = bounds.width
        firstImageView.frame = CGRect(x: 0, y: 0, width: w, height: h)
        secondImageView.frame = CGRect(x: 0, y: h, width: w, height: h)
    }
    
    @IBInspectable
    public var image: UIImage? {
        didSet {
            firstImageView.image = image
            secondImageView.image = image
            if let image = image {
                let height = bounds.width * image.size.height / image.size.width
                let size = CGSize(width: bounds.width, height: height)
                firstImageView.frame.size = size
                secondImageView.frame.size = size
                layoutIfNeeded()
            }
        }
    }
    
    @IBInspectable
    public var duration: TimeInterval = 20.0
    
    public func startAnimation() {
        DispatchQueue.main.async {
            let pageHeight = self.firstImageView.frame.height
            if pageHeight <= 0 {
                return
            }
            UIView.animate(withDuration: self.duration, delay: 0, options: [.curveLinear], animations: { [weak self] in
                self?.firstImageView.frame.origin.y = -pageHeight
                self?.secondImageView.frame.origin.y = 0
            }) { [weak self] _ in
                guard let strongSelf = self else {
                    return
                }
                strongSelf.firstImageView.frame.origin.y = pageHeight
                strongSelf.secondImageView.frame.origin.y = 0
                (strongSelf.firstImageView, strongSelf.secondImageView) = (strongSelf.secondImageView, strongSelf.firstImageView)
                strongSelf.startAnimation()
            }
        }
    }
}
```

有兴趣的可以尝试用 UICollectionView的方式实现。