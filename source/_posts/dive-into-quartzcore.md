---
title: Dive Into QuartzCore
comments: false
date: 2025-12-30 21:51:34
tags: iOS
---
本文主要介绍了 iOS 开发中涉及到的动画底层框架 QuartzCore

<!-- more -->

## 什么是 QuartzCore

QuartzCore 包含 Core Animation 和 Core Image，Apple 把二者合并成QuartzCore.framework

QuartzCore 是负责 CALayer 的管理、协同 Core Graphics 渲染 Layer 内容、以及实现 Animation 的一个库；早期 Apple 内部代号 LayerKit（来源路边），本文主要着重介绍 Core Animation 的一些内部实现

*LayerKit 这个名字暗示了 QuartzCore 的主要功能，虽然找不到来源了，可以辅助大家理解*

## Core Animation

Core Animation 是 Apple 提供给我们用于视图动画的基础框架，同时也是整个 App 底层渲染的核心组件，目的是提供更好的性能和支持动画化视图内容

![](ca_architecture_2x.png)

### CALayer

Core Animation 的核心是 CALayer，视图的管理和渲染以及动画的实现都围绕 CALayer 展开，CALayer 不直接定义自己的外观，而是负责管理一个 Bitmap 的状态信息。这些状态信息可以被视作一个 Model Object，这个概念后文也会展开说明

### CALayer 管理 bitmap 和相关状态信息

CALayer 有一个名为 contents 的 Any 属性 `var contents: Any? { get set }`，实际工程中我们可以通过这个属性来改变 Layer 的展示，如下面代码所示；需要注意的是，虽然这个属性被定义成 Any，但只有当我们赋值一个 CGImage 时 Layer 才能够正确渲染​

```Swift
import UIKit

class ViewController: UIViewController {

    override func viewDidLoad() {
        super.viewDidLoad()
        let image = UIImage(named: "Image")
        view.layer.contents = image?.cgImage
    }

}
```

实际工程中 QuartzCore 内部并不会将视图内容渲染为 CGImage，有相当一个部分视图会被渲染为 CABackingStore 或者一些 QuartzCore 的内部类，这些内部类的实现上类似 Metal/OpenGL 的纹理，QuartzCore 通过 GPU 能够加速这一部分渲染

另外一种渲染自定义 contents 的方法是通过 CAMetalLayer 的 `nextDrawable()` 获取到纹理信息，然后通过 Metal Shader 进行渲染自定义的问题，但这又是另一个庞大的主题了

### CALayer 管理渲染结果，渲染过程由 Delegate 实现

实际工程中，QuartzCore 会利用 Core Graphics 和 Metal 来渲染视图的内容，但由于这一部分的实现都是 Apple 内部的实现，我们也只能从有限的信息里来推测实际的工作方式，需要注意的是这里用词上的区分，CALayer 本身并不负责渲染内容，而更多是负责管理已经被绘制好的 Bitmap 以及相关的状态信息

```Swift
// CALayer 通过 Delegate 实现渲染
// CALayerDelegate
optional func display(_layer: CALayer)
optional func draw(
    _ layer: CALayer,
    in ctx: CGContext
)
```

在一部分 CALayer 的子类中，QuartzCore 允许我们通过 CALayerDelegte 来自定义渲染的流程。下面的代码中定义了一个 CATextLayer ，并且使用了一个空实现的 CALayerDelegate 替换掉了CATextLayer 的默认渲染行为，随后我们借助 Core Text 实现这个文本的渲染

未完待续