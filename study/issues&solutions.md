# 开发中一些问题罗列及解决办法（持续更新ing）

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