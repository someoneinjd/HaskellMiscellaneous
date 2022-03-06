为了让类型签名更加易懂，很多时候我们会用type为现有的类型起个别名

```haskell
type Position = (Int,Int)
```

emm，一个在平面上的坐标，用2个整数表示，非常显然。但是如果不写类型标注，你就看不出二元组到底是 `Position` 还是用于其他用途的二元组

```haskell
mkPos :: Int -> Int -> Position
mkPos x y = (x, y)
-- mkPos = (,)
```

这样呢？我们用抽象数据类型的方法，弄出一个构造子来，这样构造 `Position` 是很明显了。但是，还不够，这样的类型约束毕竟还是弱的，为了让这一切更加明确，我们使用 `newtype`

```haskell
newtype Position = Position (Int,Int)
```

以上，这是一个好方案，如果你忘记了显式调用 `Position`，类型检查不会轻易放过你，假如你有需要复用 `(Int,Int)`，那 `newtype` 是避免混淆的好方法，举个例子，扫雷！

我们在一个 9×9 的 Field 上进行游戏，同时对每个格子，用 `Position` 类型表达它们的位置。同时，我们使用一个非常简单的方法放置地雷，任取一个随机数并且取n的模，余数会严格限定在 [0, n) 之间。同时随机数本身也可用很简单的方式解决，一个方程和一个初始种子，再加上状态就行了。

```haskell
newtype State = State (Int,Int)
```

我承认 `System.Random` 和 `State  Monad` 的存在使得我举的例子有点傻气，但是我也不是一个特别聪明的人，而且我真的这样做。总之 `newtype` 的封装使得同一类型可以安全地用于不同概 念的抽象，并且，`newtype` 定义的“新类型”实际上在运行时没有多余成本。该是二元组的还是二元组。

更“好”的例子：`Data.Monoid` 模块中的 `Sum` 和 `Product` 类型

参考：[GHC/Coercible - HaskellWiki](https://wiki.haskell.org/GHC/Coercible)

给出一个 `newtype` 定义

```haskell
newtype HTML = MkHTML String

-- 在 HTML 类型和 String 之间的转换非常简单。

toHTML :: String -> HTML
toHTML s = MkHTML s

fromHTML :: HTML -> String
fromHTML (MkHTML s) = s

-- 这两个函数都是没有运行时负担的。

-- 但是要如何转换出一整个列表的 HTML 呢? 我们可以试着写出转换函数

toHTMLs :: [String] -> [HTML]
toHTMLs = map MkHTML


-- 不妙了，虽然 MkHTML 啥都没干，但是 map 对原 list 的遍历和重建副本是有代价的。

-- 解决方案是 GHC7.8.1 引入的 Coercible

import Data.Coerce
toHTMLs :: [String] -> [HTML]
toHTMLs = coerce

-- 这是 GHC 留下的一个类型系统上的后门，由 GHC 本身实现，但是妙处在于，虽然它 hack 了类型系统，但是它的类型转换是安全的。

-- 论文：http://research.microsoft.com/en-us/um/people/simonpj/papers/ext-f/coercible.pdf

-- 太长不看版：

-- newtype 定义的封装可以与原类型任意使用 coerce 转换，无运行时成本。(但是，这里没有讨论带类型参数的情况)

-- 对于使用了所谓“Phantom Type”的 newtype 定义，coerce 也能识别

newtype NT a = MkNT ()

trans :: NT Bool -> NT Int
trans = coerce

-- 对于藏在其他结构内部的 newtype 定义，当然 也没问题。

newtype Age = MkAge Int
newtype AgeRange = MkAR (Int,Int)
newtype BigAge = MkBig Age

-- (<=>)意为可用 coerce 转换

-- 以下都没问题

-- Either Int Age <=> Either Int Int
-- (Int -> Age) <=> (Age -> Int)
-- Map k Age <=> Map k Int
-- (Age,Age) <=> AgeRange <=> (Int,Int)

-- 具体的实现和规则可见论文。
```