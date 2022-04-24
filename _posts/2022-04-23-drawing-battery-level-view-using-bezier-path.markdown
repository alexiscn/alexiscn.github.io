---
layout: post
title:  "用 UIBezierPath 绘制iOS系统电量"
date:   2022-04-23 15:30:10 +0800
tag: iOS
---

最近给 [Filebox](https://filebox.live) 的播放器新增了显示当前设备电量的功能，记录一下。

![](/assets/images/2022/battery.75.png)

## 绘制电量

上图是 SF Symbols 中的电量图标，初步可以拆解为

- 外部圆角矩形边框
- 内部圆角矩形填充
- 右侧扇形

```swift
override func draw(_ rect: CGRect) {
    super.draw(rect)

    UIColor.clear.setFill()
    
    // draw outline
    let outRect = bounds.insetBy(dx: 6, dy: 7)
    let outPath = UIBezierPath(roundedRect: outRect, cornerRadius: 3)
    outPath.lineWidth = 1.5
    UIColor.gray.setStroke()
    outPath.stroke()

    // draw inner rectangle
    let innerFullRect = outRect.insetBy(dx: 2, dy: 2)
    let innerRect = CGRect(x: innerFullRect.origin.x,
                            y: innerFullRect.origin.y,
                            width: innerFullRect.width * CGFloat(batterLevel),
                            height: innerFullRect.height)
    let innerPath = UIBezierPath(roundedRect: innerRect, cornerRadius: 1)
    fillColor.setFill()
    innerPath.fill()
}
```

绘制出来的效果如下图所示，现在还差扇形区域

![](/assets/images/2022/battery2.png)


右侧扇形区域可以通过clip的方式实现，具体代码如下

```swift

override func draw(_ rect: CGRect) {

    // ...

    // draw right ellipse
    if let context = UIGraphicsGetCurrentContext() {
        let rect = CGRect(x: bounds.width - 6, y: (bounds.height - 4)/2.0, width: 4, height: 4)
        context.saveGState()
        context.setFillColor(UIColor.gray.cgColor)
        
        let clipRect = rect.offsetBy(dx: 2, dy: 0)
        context.clip(to: clipRect)
        context.fillEllipse(in: rect)
        context.restoreGState()
    }
}

```

![](/assets/images/2022/battery3.png)

现在基本的轮廓已经画出来了，只剩下完善具体的细节逻辑

- 电量颜色
- 是否正在充电

## 监听电量变化

在电量发生变化时需要重新绘制。

```swift

func commonInit() {
    NotificationCenter.default.addObserver(self, selector: #selector(handleBatteryChanged), name: UIDevice.batteryLevelDidChangeNotification, object: nil)
}


@objc private func handleBatteryChanged() {
    setNeedsDisplay()
}

```

## 监听充电状态变化

监听电量变化还不够，还需要监听当前设备是否在充电状态，以及是否开启低电量模式。

```swift
func commonInit() {
    UIDevice.current.isBatteryMonitoringEnabled = true
    NotificationCenter.default.addObserver(self, selector: #selector(handleBatteryChanged), name: UIDevice.batteryLevelDidChangeNotification, object: nil)
    NotificationCenter.default.addObserver(self, selector: #selector(handleBatteryChanged), name: UIDevice.batteryStateDidChangeNotification, object: nil)
}

deinit {
    UIDevice.current.isBatteryMonitoringEnabled = false
}

```

## 完整代码

完整代码如下，里面写死了部分大小

```swift

import UIKit
import Foundation

/// let batteryView = BatteryLevelView(frame: CGRect(x: 0, y: 0, width: 32, height: 24))
class BatteryLevelView: UIView {
    
    var lightingImage: UIImage?
    
    override init(frame: CGRect) {
        super.init(frame: frame)
        commonInit()
    }
    
    required init?(coder: NSCoder) {
        super.init(coder: coder)
        commonInit()
    }
    
    deinit {
        UIDevice.current.isBatteryMonitoringEnabled = false
    }
    
    private func commonInit() {
        lightingImage = UIImage(named: "lighting_12x12_")
        backgroundColor = .clear
        UIDevice.current.isBatteryMonitoringEnabled = true
        NotificationCenter.default.addObserver(self, selector: #selector(handleBatteryChanged), name: UIDevice.batteryLevelDidChangeNotification, object: nil)
        NotificationCenter.default.addObserver(self, selector: #selector(handleBatteryChanged), name: UIDevice.batteryStateDidChangeNotification, object: nil)
        NotificationCenter.default.addObserver(self, selector: #selector(handleBatteryChanged), name: .NSProcessInfoPowerStateDidChange, object: nil)
    }
    
    @objc private func handleBatteryChanged() {
        setNeedsDisplay()
    }
    
    override func draw(_ rect: CGRect) {
        super.draw(rect)
        
        let batterLevel = abs(UIDevice.current.batteryLevel)
        let isCharging = UIDevice.current.batteryState != .unplugged
        let isLowPowerModeEnabled = ProcessInfo.processInfo.isLowPowerModeEnabled
        
        UIColor.clear.setFill()
        
        // draw outline
        let outRect = bounds.insetBy(dx: 6, dy: 7)
        let outPath = UIBezierPath(roundedRect: outRect, cornerRadius: 3)
        outPath.lineWidth = 1.5
        UIColor.gray.setStroke()
        outPath.stroke()
        
        // draw inner rectangle
        let fillColor: UIColor
        if isLowPowerModeEnabled {
            fillColor = UIColor(hexString: "#F9D74A")
        } else if isCharging {
            fillColor = UIColor.green
        } else if batterLevel > 0 && batterLevel < 0.2 {
            fillColor = UIColor.red
        } else {
            fillColor = UIColor.white
        }
        fillColor.setFill()
        
        let innerFullRect = outRect.insetBy(dx: 2, dy: 2)
        let innerRect = CGRect(x: innerFullRect.origin.x,
                               y: innerFullRect.origin.y,
                               width: innerFullRect.width * CGFloat(batterLevel),
                               height: innerFullRect.height)
        let innerPath = UIBezierPath(roundedRect: innerRect, cornerRadius: 1)
        fillColor.setFill()
        innerPath.fill()
                
        // draw right ellipse
        if let context = UIGraphicsGetCurrentContext() {
            let rect = CGRect(x: bounds.width - 6, y: (bounds.height - 4)/2.0, width: 4, height: 4)
            context.saveGState()
            context.setFillColor(UIColor.gray.cgColor)
            
            let clipRect = rect.offsetBy(dx: 2, dy: 0)
            context.clip(to: clipRect)
            context.fillEllipse(in: rect)
            context.restoreGState()
        }
        
        // draw lighting indicator
        if isCharging {
            let imageRect = CGRect(x: (bounds.width - 12)/2.0, y: (bounds.height - 12)/2.0, width: 12, height: 12)
            lightingImage?.draw(in: imageRect)
        }
    }
}
```