---
layout: post
title: Halogen-04-执行Effects
categories:
- 前端 purescript halogen
tags:
- 前端 purescript halogen
author:
  name: purescript-halogen
  link: https://github.com/purescript-halogen/purescript-halogen/blob/master/docs/guide/03-Performing-Effects.md
date: 2022-01-02 11:04 +0800
---

### Performing Effects

到目前为止，我们已经涵盖了很多领域。您知道如何编写`Halogen HTML`。您可以定义响应用户交互的组件并按类型对组件的每个部分进行建模, 有了这个基础，我们就可以在编写应用程序时继续使用另一个重要的工具: 执行`effects`。

在本章中，我们将通过两个示例探索如何在您的组件中执行`effects`: 生成随机数并发出`HTTP`请求。一旦您知道如何执行`effects`，您就可以很好地掌握`Halogen`基础知识.

在我们开始之前，重要的是要知道您只能在评估期间执行`effects`，例如像`handleAction`这样使用`HalogenM`类型的函数. 但是您无法在生成初始状态或渲染期间执行`effects`。由于您只能在`HalogenM`中执行`effects`，因此在深入研究示例之前，让我们简要了解一下它的更多信息。

#### HalogenM类型

如果您还记得上一章的内容，`handleAction`函数会返回一个名为`HalogenM`的类型。这是我们写的`handleAction`:

```purescript
handleAction :: forall output m. Action -> HalogenM State Action () output m Unit
```

`HalogenM`是`Halogen`的关键部分，通常称为`eval`单子。这个`monad`启用了`Halogen`特性，如状态、分叉线程、开始订阅等。但它非常有限，仅与`Halogen`特定的性质有关。事实上，`Halogen`组件没有内置的`effects`机制！

相反，`Halogen`允许您选择要在组件中与`HalogenM`一起使用的`monad`。您可以访问`HalogenM`的所有功能以及您选择的`monad`支持的任何功能. 这用类型参数`m`表示，它代表`monad`。

仅使用`Halogen`特定性质的组件可以让此类型参数保持打开状态。例如，我们的计数器只更新状态。但是执行`effects`的组件可以使用 `Effect`或`Aff monad`，或者您可以提供自己的自定义`monad`。

这个`handleAction`可以使用来自`HalogenM`的函数，比如`modify_`，也可以使用来自`Effect`的`effectful`函数:
```purescript
handleAction :: forall output. Action -> HalogenM State Action () output Effect Unit
```

这个可以使用来自`HalogenM`的函数以及来自`Aff`的`effectful`函数:
```purescript
handleAction :: forall output. Action -> HalogenM State Action () output Aff Unit
```

在`Halogen`中更常见的是对类型参数`m`使用约束来描述`monad`可以做什么，而不是选择特定的`monad`, 这允许您随着应用程序的增长将多个`monad`混合在一起。例如，大多数`Halogen`应用程序将通过以下类型签名使用来自`Aff`的函数:

```purescript
handleAction :: forall output m. MonadAff m => Action -> HalogenM State Action () output m Unit
```

这让您可以完成硬编码`Aff`类型所做的一切，但也可以让您混合其他约束。

最后一件事: 当你为你的组件选择一个`monad`时，它会出现在你的`HalogenM`类型、你的`Component`类型中，如果你正在使用子组件，那么它会出现在你的`ComponentHTML`类型中:

```purescript
component :: forall query input output m. MonadAff m => H.Component query input output m

handleAction :: forall output m. MonadAff m => Action -> HalogenM State Action () output m Unit

-- We aren't using child components, so we don't have to use the constraint here, but
-- we'll learn about when it's required in the parent & child components chapter.
render :: forall m. State -> H.ComponentHTML Action () m
```

### 一个Effect例子: 随机数

让我们创建一个新的简单组件，每次单击按钮时都会生成一个新的随机数。在您阅读示例时，请注意它如何使用与我们用于编写计数器的相同类型和函数。随着时间的推移，您将习惯于快读`Halogen`组件的状态、动作和其他类型，以了解其功能的要点，并熟悉标准函数，如`initialState`、`render` 和 `handleAction`。

> 您可以将此示例粘贴到[Try Purescript](https://try.purescript.org/)中以交互方式探索它。您还可以在此存储库的示例目录中查看并运行[完整的示例代码](https://github.com/purescript-halogen/purescript-halogen/tree/master/examples/effects-effect-random)。


请注意，我们没有在我们的`initialState`或`render`函数中执行任何`effects` -- 例如，我们将我们的状态初始化为`Nothing`而不是为我们的初始状态生成一个随机数 —— 但是我们可以在我们的`handleAction`函数(使用`HalogenM`类型)中自由地执行`effects`。

```purescript
module Main where

import Prelude

import Data.Maybe (Maybe(..), maybe)
import Effect (Effect)
import Effect.Class (class MonadEffect)
import Effect.Random (random)
import Halogen as H
import Halogen.Aff (awaitBody, runHalogenAff)
import Halogen.HTML as HH
import Halogen.HTML.Events as HE
import Halogen.VDom.Driver (runUI)

main :: Effect Unit
main = runHalogenAff do
  body <- awaitBody
  runUI component unit body

type State = Maybe Number

data Action = Regenerate

component :: forall query input output m. MonadEffect m => H.Component query input output m
component =
  H.mkComponent
    { initialState
    , render
    , eval: H.mkEval $ H.defaultEval { handleAction = handleAction }
    }

initialState :: forall input. input -> State
initialState _ = Nothing

render :: forall m. State -> H.ComponentHTML Action () m
render state = do
  let value = maybe "No number generated yet" show state
  HH.div_
    [ HH.h1_
        [ HH.text "Random number" ]
    , HH.p_
        [ HH.text ("Current value: " <> value) ]
    , HH.button
        [ HE.onClick \_ -> Regenerate ]
        [ HH.text "Generate new number" ]
    ]

handleAction :: forall output m. MonadEffect m => 
    Action -> 
    H.HalogenM State Action () output m Unit
handleAction = case _ of
  Regenerate -> do
    newNumber <- H.liftEffect random
    H.modify_ \_ -> Just newNumber
```

如您所见，执行`effects`的组件与不执行`effects`的组件没有太大区别！我们只做了两件事:

* 我们为组件和`handleAction`函数的`m`类型参数添加了`MonadEffect`约束. 我们的`render`函数不需要约束，因为我们没有任何子组件。
* 我们实际上第一次使用了一个`effect`: `random`函数，它来自`Effect.Random`。

让我们再分解一下这个`effect`。
```purescript
--                          [1]
handleAction :: forall output m. MonadEffect m => 
        Action -> 
        H.HalogenM State Action () output m Unit
handleAction = case _ of
  Regenerate -> do
    newNumber <- H.liftEffect random -- [2]
    H.modify_ \_ -> Just newNumber   -- [3]
```

* 我们已经限制了我们的`m`类型参数说我们支持任何`monad`，只要那个`monad`支持`MonadEffect`. 这是"我们需要能够在我们的评估代码中使用`Effect`函数"的另一种说法。
* 随机函数的类型为`Effect Number`。但是我们不能直接使用它：我们的组件不支持`Effect`而是支持任何`monad m`，只要该`monad`可以从`Effect`运行`effects`. 这是一个细微的区别，但最终我们要求`random`函数的类型为`MonadEffect m => m Number`而不是直接为 `Effect`. 幸运的是，我们可以使用`LiftEffect`函数将任何`Effect`类型转换为`MonadEffect m => m`。这是`Halogen`中的常见模式，因此如果您使用`MonadEffect`，请记住`liftEffect`。
* `modify_`函数让你更新状态，它直接来自带有其他状态更新功能的`HalogenM`。在这里，我们使用它来将新的随机数写入我们的状态。

这是一个很好的示例，说明您可以如何自由地将`Effect`中的`effects`与特定于`Halogen`的函数(如`modify_`)交织在一起。让我们再做一次，这次使用`Aff monad`来实现异步效果。

### 一个Aff例子: HTTP Requests

从`Internet`上的其他地方获取信息是很常见的。例如，假设我们想使用`GitHub`的`API`来获取用户。我们将使用[affjax](https://pursuit.purescript.org/packages/purescript-affjax)包来发出我们的请求, 它本身依赖于`Aff monad`来实现异步效果。

不过，这个例子更有趣: 我们还将使用`preventDefault`函数来防止表单提交刷新页面，该页面在`Effect`中运行。这意味着我们的示例展示了如何将不同的`effects`(`Effect`和`Aff`)与`Halogen`函数(`HalogenM`)交织在一起。

> 与`Random`示例一样，您可以将此示例粘贴到[Try Purescript](https://try.purescript.org/)中以交互方式探索它。您还可以在此存储库的`examples`目录中查看并运行[完整的示例代码](https://github.com/purescript-halogen/purescript-halogen/tree/master/examples/effects-aff-ajax).

这个组件定义应该看起来很熟悉。我们定义我们的`State`和`Action`类型并实现我们的`initialState`、`render`和`handleAction`函数. 我们将它们组合到我们的组件规范中，并将它们变成一个有效的组件`H.mkComponent`。

再次注意，我们的`effects`集中在`handleAction`函数中，并且在构造初始状态或渲染`Halogen HTML`时没有执行任何`effects`。

```purescript
module Main where

import Prelude

import Affjax as AX
import Affjax.ResponseFormat as AXRF
import Data.Either (hush)
import Data.Maybe (Maybe(..))
import Effect (Effect)
import Effect.Aff.Class (class MonadAff)
import Halogen as H
import Halogen.Aff (awaitBody, runHalogenAff)
import Halogen.HTML as HH
import Halogen.HTML.Events as HE
import Halogen.HTML.Properties as HP
import Halogen.VDom.Driver (runUI)
import Web.Event.Event (Event)
import Web.Event.Event as Event

main :: Effect Unit
main = runHalogenAff do
  body <- awaitBody
  runUI component unit body

type State =
  { loading :: Boolean
  , username :: String
  , result :: Maybe String
  }

data Action
  = SetUsername String
  | MakeRequest Event

component :: forall query input output m. MonadAff m => H.Component query input output m
component =
  H.mkComponent
    { initialState
    , render
    , eval: H.mkEval $ H.defaultEval { handleAction = handleAction }
    }

initialState :: forall input. input -> State
initialState _ = { loading: false, username: "", result: Nothing }

render :: forall m. State -> H.ComponentHTML Action () m
render st =
  HH.form
    [ HE.onSubmit \ev -> MakeRequest ev ]
    [ HH.h1_ [ HH.text "Look up GitHub user" ]
    , HH.label_
        [ HH.div_ [ HH.text "Enter username:" ]
        , HH.input
            [ HP.value st.username
            , HE.onValueInput \str -> SetUsername str
            ]
        ]
    , HH.button
        [ HP.disabled st.loading
        , HP.type_ HP.ButtonSubmit
        ]
        [ HH.text "Fetch info" ]
    , HH.p_
        [ HH.text $ if st.loading then "Working..." else "" ]
    , HH.div_
        case st.result of
          Nothing -> []
          Just res ->
            [ HH.h2_
                [ HH.text "Response:" ]
            , HH.pre_
                [ HH.code_ [ HH.text res ] ]
            ]
    ]

handleAction :: forall output m. MonadAff m => Action -> H.HalogenM State Action () output m Unit
handleAction = case _ of
  SetUsername username -> do
    H.modify_ _ { username = username, result = Nothing }

  MakeRequest event -> do
    H.liftEffect $ Event.preventDefault event
    username <- H.gets _.username
    H.modify_ _ { loading = true }
    response <- H.liftAff $ AX.get AXRF.string ("https://api.github.com/users/" <> username)
    H.modify_ _ { loading = false, result = map _.body (hush response) }
```

这个例子特别有趣，因为:

* 它混合了来自多个`monad`的函数(`preventDefault`是`Effect`，`AX.get`是`Aff`，`gets`和`modify_`是`HalogenM`)。我们可以使用`liftEffect`和`liftAff`以及我们的约束来确保一切都很好地协同工作。
* 我们只有一个约束，`MonadAff`。那是因为任何可以在`Effect`中运行的东西也可以在`Aff`中运行，所以`MonadAff`意味着 `MonadEffect`。
* 我们正在一次评估中进行多个状态更新。

最后一点特别重要：当您修改组件呈现的状态时。这意味着在本次评估期间，我们:
* 将`loading`设置为`true`,这会导致组件重新渲染并显示`Working..`
* 将`loading`设置为`false`并更新结果，这会导致组件重新渲染并显示结果(如果有的话)。


值得注意的是，因为我们使用的是`MonadAff`，所以我们的请求不会阻止组件做其他工作，而且我们不必处理回调来获得这种异步超能力。我们在`MakeRequest`中编写的计算只是暂停，直到我们得到响应，然后继续第二次更新状态。

仅在必要时修改状态并在可能的情况下一起批量更新是一个聪明的主意(就像我们调用`modify_`一次来更新`loading`和`result`字段一样). 这有助于确保您只在需要时重新渲染。

### 重新审视事件处理

在这个例子中发生了很多事情，所以值得花点时间关注它引入的新事件处理特性。与按钮示例的简单点击处理程序不同, 此处定义的处理程序确实使用了它们所提供的事件数据:
* 用户名输入的值由`onValueInput`处理程序(`SetUsername`操作)使用。
* 在`onSubmit`处理程序(`MakeRequest`操作)中的事件上调用`preventDefault`。


传递给处理程序的参数类型取决于用于附加它的函数。有时，对于`onValueInput`，处理程序只是接收从事件中提取的数据 - 在这种情况下是字符串。大多数其他`on...`函数设置一个处理程序来接收整个事件，或者作为`Event`类型的值，或者作为像`MouseEvent`这样的特殊类型。详细信息可以在[Halogen.HTML.Events](https://pursuit.purescript.org/packages/purescript-halogen/docs/Halogen.HTML.Events)的模块文档中找到；用于事件的类型和函数可以在[web-events](https://pursuit.purescript.org/packages/purescript-web-events)和[web-uievents](https://pursuit.purescript.org/packages/purescript-web-uievents)包中找到。


