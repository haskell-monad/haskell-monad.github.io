---
layout: post
title: Halogen-02-渲染HalogenHTML
categories:
- 前端 purescript halogen
tags:
- 前端 purescript halogen
author:
  name: purescript-halogen
  link: https://github.com/purescript-halogen/purescript-halogen/blob/master/docs/guide/01-Rendering-Halogen-HTML.md
date: 2022-01-02 11:04 +0800
---

### 渲染Halogen HTML

`Halogen HTML`元素是`Halogen`应用程序的最小构建块。这些元素描述了您希望在屏幕上看到的内容。
`Halogen HTML`元素不是组件（我们将在下一章中介绍组件），如果没有组件，元素就无法呈现。但是，编写生成`Halogen HTML`的辅助函数然后在组件中使用这些函数是很常见的。

在本章中，我们将探索在没有组件或事件的情况下编写`HTML`。


#### Halogen HTML
您可以使用`Halogen.HTML`或`Halogen.HTML.Keyed`模块中的函数编写`Halogen HTML`，如下例所示：

```purescript
import Halogen.HTML as HH

element = HH.h1 [ ] [ HH.text "Hello, world" ]
```

`Halogen HTML`元素可以被认为是浏览器`DOM`元素, 但它们由`Halogen`库控制，而不是`DOM`中的实际元素。在幕后，`Halogen`负责更新实际`DOM`以匹配您编写的代码。


`Halogen`中的元素接受两个参数：

* 应用于元素的`attributes`、`properties`、`event handlers`和/或`references`的数组.这些对应于普通的`HTML`属性(如`placeholder`)和事件处理程序(如`onClick`)。我们将在下一章学习如何处理事件，本章只关注属性。
* 子元素数组，如果该元素支持子元素。

举个简单的例子，让我们把这个普通的`HTML`翻译成`Halogen HTML`：
```html
<div id="root">
  <input placeholder="Name" />
  <button class="btn-primary" type="submit">
    Submit
  </button>
</div>
```
让我们分解我们的`Halogen HTML`：
* 我们的`Halogen`代码具有与普通`HTML`相同的形状：一个`div`包含一个`input`和一个`button`，`button`本身包含纯文本。
* 属性从标签内的键值对移动到元素的属性数组中.
* 如果元素支持子元素，则子元素从标签内移动到子元素数组。

```purescript
import Halogen.HTML as HH
import Halogen.HTML.Properties as HP

html =
  HH.div
    [ HP.id "root" ]
    [ HH.input
        [ HP.placeholder "Name" ]
    , HH.button
        [ HP.classes [ HH.ClassName "btn-primary" ]
        , HP.type_ HP.ButtonSubmit
        ]
        [ HH.text "Submit" ]
    ]
```

您可以在此处看到`Halogen`对类型安全的重视:

* `text input`不能有子元素，因此`Halogen`不允许该元素将更多元素作为参数。
* `button`的`type`属性只能使用某些值，因此`Halogen`使用`sum`类型来限制它们。
* `CSS classes`使用`ClassName newtype`，以便在需要时可以对其进行特殊处理；例如，`classes`函数确保您的`classes`在组合时以空格分隔。

一些`HTML`元素和属性与`PureScript`中的保留关键字或`Prelude`中的常见函数发生冲突，因此`Halogen`为它们添加了下划线。这就是为什么您在上面的示例中看到`type_`而不是`type`的原因。

当您不需要在`Halogen HTML`元素上设置任何属性时，您可以改用其带下划线的版本。例如，下面的`div`和`button`元素没有属性：
```purescript
html = HH.div [ ] [ HH.button [ ] [ HH.text "Click me!"] ]
```

这意味着我们可以使用它们的下划线版本重写它们。这有助于保持`HTML`整洁:
```purescript
html = HH.div_ [ HH.button_ [ HH.text "Click me!" ] ]
```

#### 在 Halogen HTML 中编写函数

为`Halogen HTML`编写辅助函数是很常见的。由于`Halogen HTML`是由普通的`PureScript`函数构建的，因此您可以在代码中自由穿插其他函数。

在这个例子中，我们的函数接受一个整数并将其呈现为文本:
```purescript
header :: forall w i. Int -> HH.HTML w i
header visits = 
  HH.h1_ 
    [ HH.text $ "You've had " <> show visits <> " visitors" ] 
```

我们还可以渲染事物列表:

```purescript
lakes = [ "Lake Norman", "Lake Wylie" ]

html :: forall w i. HH.HTML w i
html = HH.div_ (map HH.text lakes)
-- same as: HH.div_ [ HH.text "Lake Norman", HH.text "Lake Wylie" ]
```

这些函数引入了一种新类型`HH.HTML`，这是您以前从未见过的。别担心！这是`Halogen HTML`的类型，我们将在下一节中了解它。现在，让我们继续学习在`HTML`中使用函数。

一个常见的要求是有条件地呈现一些`HTML`. 你可以用普通的`if`和`case`语句来做到这一点，但为常见模式编写辅助函数很有用. 让我们来看看您可能在自己的应用程序中编写的两个辅助函数，这将帮助我们获得更多使用`Halogen HTML`编写函数的练习。

首先，您有时可能需要处理可能存在或不存在的元素。像下面这样的函数可以让你渲染一个存在的值，否则渲染一个空节点。

```purescript
maybeElem :: forall w i a. Maybe a -> (a -> HH.HTML w i) -> HH.HTML w i
maybeElem val f =
  case val of
    Just x -> f x
    _ -> HH.text ""

-- 如果name有的话, 则渲染它
renderName :: forall w i. Maybe String -> HH.HTML w i
renderName mbName = maybeElem mbName \name -> HH.text name
```

其次，您可能只想在条件为真时呈现一些`HTML`，如果条件不符合，则不计算`HTML`。您可以通过将其评估隐藏在函数后面来实现此目的，这样`HTML`仅在条件为真时才计算。

```purescript
whenElem :: forall w i. Boolean -> (Unit -> HH.HTML w i) -> HH.HTML w i
whenElem cond f = if cond then f unit else HH.text ""

-- 渲染旧的number，但前提是它与新的number不同
renderOld :: forall w i. { old :: Number, new :: Number } -> HH.HTML w i
renderOld { old, new } = 
  whenElem (old /= new) \_ -> 
    HH.div_ [ HH.text $ show old ]
```

现在我们已经探索了几种使用`HTML`的方法，让我们了解更多关于描述它的类型。

#### HTML 类型

到目前为止，我们已经编写了没有类型签名的`HTML`。但是当您在应用程序中编写`Halogen HTML`时，您将包含类型签名。

##### HTML w i
`HTML`是`Halogen`中`HTML`的核心类型,它用于不绑定到特定类型组件的`HTML`元素. 例如，它被用作我们目前看到的`h1`、`text`和`button`元素的类型。您也可以在定义自己的自定义`HTML`元素时使用此类型。

`HTML`类型有两个类型参数：`w`，代表`widget`，描述可以在`HTML`中使用哪些组件；`i`，代表`input`，代表用于处理 `DOM`事件的类型。

当您为`Halogen HTML`编写不需要响应`DOM`事件的辅助函数时, 那么您通常会使用`HTML`类型而不指定`w`和`i`是什么。例如，这个辅助函数可以让你创建一个按钮，给定一个标签:

```purescript
primaryButton :: forall w i. String -> HH.HTML w i
primaryButton label =
  HH.button
    [ HP.classes [ HH.ClassName "primary" ] ]
    [ HH.text label ]
```

你也可以接受`HTML`作为标签，而不是只接受一个字符串:

```purescript
primaryButton :: forall w i. HH.HTML w i -> HH.HTML w i
primaryButton label =
  HH.button
    [ HP.classes [ HH.ClassName "primary" ] ]
    [ label ]
```

当然，作为一个按钮，您可能希望在单击它时执行某些操作。别担心——我们将在下一章介绍处理`DOM`事件！

##### ComponentHTML 和 PlainHTML

您通常会在`Halogen`应用程序中看到另外两种`HTML`类型.

当您编写旨在处理特定类型组件的`HTML`时，将使用`ComponentHTML`。它也可以在组件外部使用，但最常用于组件内部。我们将在下一章中了解有关此类型的更多信息。

`PlainHTML`是限制性更强的`HTML`版本，用于不包含组件且不响应`DOM`中的事件的`HTML`.该类型允许您隐藏`HTML`的两个类型参数，这在您将`HTML`作为值传递时非常方便。但是，如果要将这种类型的值与其他响应`DOM`事件或包含组件的`HTML`组合时，则需要使用`fromPlainHTML`进行转换.


##### IProp

当您从`Halogen.HTML.Properties`和`Halogen.HTML.Events`模块中查找函数时,您会看到`IProp`类型突出显示。例如，这是一个`placeholder`函数，它可以让您在文本字段上设置字符串`placeholder`属性:

```purescript
placeholder :: forall r i. String -> IProp (placeholder :: String | r) i
placeholder = prop (PropName "placeholder")
```

`IProp`类型用于事件和属性,它使用`row`类型来唯一标识特定事件和属性；当您将这些属性之一与`Halogen HTML`元素结合使用时, `Halogen`能够验证您应用该属性的元素是否实际支持它。

这是可能的，因为`Halogen HTML`元素还带有一个`row`类型，其中列出了它可以支持的所有属性和事件. 当您将属性或事件应用于元素时，`Halogen`会在`HTML`元素的`row`类型中查找它是否支持该属性或事件。

这有助于确保您的`HTML`格式良好。例如, 根据`DOM`规范，`<div>`元素不支持`placeholder`属性, 因此，如果您尝试在 `Halogen`中为`div`提供`placeholder`属性，则会出现编译时错误:

```purescript
-- ERROR: Could not match type ( placeholder :: String | r )
-- with type ( accessKey :: String, class :: String, ... )
html = HH.div [ HP.placeholder "blah" ] [ ]
```

此错误告诉您，您尝试将属性与不支持它的元素一起使用。它首先列出您尝试使用的属性，然后列出元素支持的属性。`Halogen`类型安全的另一个例子！

##### 添加缺失的属性

`HTML`是一种不断被修订的[living standard](https://html.spec.whatwg.org/multipage), `Halogen`试图跟上这些变化，但有时会落后.(如果您对我们如何自动化检测这些变化的过程有任何想法，请告诉[我们](https://github.com/purescript-halogen/purescript-halogen/issues/685))

您可能会发现`Halogen`中缺少某些属性。例如，您可以尝试编写:

```purescript
html = HH.iframe [ HP.sandbox "allow-scripts" ]
```

只收到此错误:

```purescript
Unknown value HP.sandbox
```

即使看起来应该[支持](https://pursuit.purescript.org/packages/purescript-dom-indexed/docs/DOM.HTML.Indexed#t:HTMLiframe)此属性:

```purescript
type HTMLiframe = Noninteractive (
    height :: CSSPixel, name :: String, 
    onLoad :: Event, sandbox :: String, 
    src :: String, srcDoc :: String, 
    width :: CSSPixel
)
```

解决方案是编写您自己的此缺失属性的实现:

```purescript
sandbox :: forall r i. String -> HH.IProp ( sandbox :: String | r ) i
sandbox = HH.prop (HH.PropName "sandbox")
```

然后你可以在你的`HTML`元素中使用它:

```purescript
html = HH.iframe [ sandbox "allow-scripts" ]
```

请打开一个问题或`PR`以添加此缺少的属性。这是一种为`Halogen`做出贡献的简单方法。


