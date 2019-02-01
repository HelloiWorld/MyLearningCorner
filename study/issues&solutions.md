# 开发中一些问题罗列及解决办法（持续更新ing）

## 目录
* [分割线隐藏的另一种方案]()
* 

## 分割线隐藏的另一种方案
通常隐藏分割线我们常用的做法是更换导航栏的背景图片及阴影图片，具体做法如下：

```
let navigationBar = UINavigationBar.appearance()
navigationBar.setBackgroundImage(UIImage.init(color: .white), for: .default)
// 必须设置backgroundImage才能使以下代码生效
navigationBar.shadowImage = UIImage()
```

然而有些情况下可能并不想更换导航栏背景图，阅读了[iOS开发中隐藏导航栏的分割线](https://www.jianshu.com/p/23d9bde85f13)的方案后，这里提供解决办法：

```
func hideSeperateLine() {
	if #available(iOS 11.0, *) {
	
	} else {
	    let backgroundView = self.navigationBar.subviews.first
	    let lineView = backgroundView?.subviews.first
	    lineView?.alpha = 0
	}
}

override func viewDidLayoutSubviews() {
    super.viewDidLayoutSubviews()
    // iOS 11以上导航栏层级发生了变更，且只有在这里才能找到subviews
    if #available(iOS 11.0, *) {
        let backgroundView = self.navigationBar.subviews.first
        for subview in backgroundView?.subviews ?? [] {
            if subview is UIImageView {
                subview.isHidden = true
                break
            }
        }
    }
}
```

## scrollView嵌套tableView的手势冲突解决方案？
效果如图 ![](https://upload-images.jianshu.io/upload_images/1792044-ec83d82069011023.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/372)

我们需要使用联动手势，首先我们要设置允许同时触发多个手势

```
// 允许同时触发多个手势
func gestureRecognizer(_ gestureRecognizer: UIGestureRecognizer, shouldRecognizeSimultaneouslyWith otherGestureRecognizer: UIGestureRecognizer) -> Bool {
    return true
}
```

然后我们要创建一个拖拽手势

```
private func addPanGestures() {
    // 添加联动手势
    let pan = UIPanGestureRecognizer(target: self, action: #selector(panAct(sender:)))
    pan.delegate = self
    self.view.addGestureRecognizer(pan)
}
```

监听滑动值的改变即可，这里监听的是滑动速度过快时才隐藏导航栏，瑕疵很大=_=

```
// MARK: 检测滚动手势、隐藏/显示导航栏
@objc private func panAct(sender: UIPanGestureRecognizer) {
    let velPoint = sender.velocity(in: self.view)
    if velPoint.y > 300  {
        if let parentVC = self.parent as? XXXVC {
            parentVC.setNavigationBarHidden(false, animated: true)
        }
    } else if velPoint.y < -300  {
        if let parentVC = self.parent as? XXXVC {
            parentVC.setNavigationBarHidden(true, animated: true)
        }
    }
}
```