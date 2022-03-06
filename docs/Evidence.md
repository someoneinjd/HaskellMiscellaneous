`Int`，`Bool`，`[a]`，每天编写 haskell 代码时总免不了要和这些类型打交道。连前端的函数式教程都免不了俗要提一嘴。类型检查挡住了很多傻逼兮兮的错误，救程序员于水深火热中。但是，haskell 的 `Prelude` 中有很多所谓的「partial function」，它们非常简单，也很好理解，可惜并不能对所有符合类型签名的输入都给出一个合理的输出，这样一来，面对不合法输入，这些函数只能只好 crash 了 。

显而易见空列表在其中“居功甚伟”，`head`，`tail` 都没办法应付它。`head` 只能无奈地在运行时输出一段 `Prelude.head: empty list`。没啥用的错误报告（GHC：你这是在为难我胖虎.jpg）

其实这还算是一个比较好处理的问题，只要求传入的 list 非空就行，大不了我去用 `safeHead` 和 monadic 的错误处理。更加头大的问题是，假如对于参数的性质判断本身代价就很大呢？比如，需要一个已经被排序的 list，但是每次只消耗一个元素。

检查可以通过函数完成

```haskell
isSorted :: Ord a => [a] -> Bool
isSorted [] = True
isSorted [x] = True
isSorted (x:y:s) = if x <= y then isSorted  (y:s) else False
```

这种返回 `Bool` 的函数一般称为「谓词」。

当然可以用 RecursiveGo 这个模式来避免每次递归都要检查一次，但是对于一个非常大的 list 来说，检查的代价简直没法忍受。C 程序员可能会说：我相信将来使用我的代码的人拥有充分的智慧，因此我无需多嘴多舌。毕竟自由的代价是永远保持警惕嘛！（高情商：相信用户；低情商：检查代价不可接受，直接开摆）

虽然我们中的大部分人写出的 haskell 代码都是 toy 性质的（你再骂.jpg），基本不用考虑这个问题。但是人总是要有点理想的嘛，万一哪天自己写的 haskell 代码被别人用了呢。这就是 Evidence 模式起作用的时候了。

```haskell
module SortedList (sortT, unSL) where

import Data.List (sort)
import Data.Coerce

newtype SortedList a = SortedList [a]

sortT :: Ord a => [a] -> SortedList a
sortT = SortedList . sort

unSL :: SortedList a -> [a]
unSL = coerce
```

这个方案结合了 NewType 和 SmartConstructor 模式， 虽然非常粗浅，但是通过 `SortedList` 这个新类型与合理的模块设计，我们可以在编译期拦截住不合理的输入并且避免了几乎所有的运行时开销！

我对 Evidence 模式的理解是，对于那些用谓词检查参数是否合适的场合，总是可以定义一个新类型存放合适的参数，上面这个例子很特殊，因为所有 list 都可以被处理为有序的 list。在有些情况下，相应类型对应的值可以被谓词分为两类，比如在 `Int` 中区分质数与非质数。

```haskell
newtype Prime = Prime Int -- INVARIANT: must be prime

prime :: Int -> Maybe Prime
```

对于不合适的值简单粗暴的返回 `Nothing` 就OK。在要消耗值的函数内可以用 case 进行模式匹配或者使用 `MaybeT`。如果想更直接一点，就用 `PatternGuards` 这个扩展。介绍在此：[Pattern and Guard Extensions](https://www.schoolofhaskell.com/school/to-infinity-and-beyond/pick-of-the-week/guide-to-ghc-extensions/pattern-and-guard-extensions%23patternguards)

Evidence 模式的另一优点是增加代码可读性。

```haskell
-- Merge two sorted list into a new sorted list.
merge :: Ord a => SortedList a -> SortedList a -> SortedList a
```

但是等等，万一有人这样写代码怎么办？

```haskell
add :: (a -> Maybe Int) -> (a -> Maybe Int) -> a -> Maybe Int
add f g x =
    if isNothing (f x) || isNothing (g x)
    then Nothing
    else Just (fromJust (f x) + fromJust (g x))
```

原作者非常无奈地写下了这段话：

> Unfortunately, even if you follow the Evidence pattern in types, you still can misuse it in values.

误用是没法避免的，再 NB 的类型系统和语言也架不住用户瞎 JB 乱搞，防君子不防小人了属于是。

Evidence 模式通过类型为一些值赋予额外的性质信息，可以在很多场景下代替谓词。思路类似的模式还有 Phantom Type。也可以说，Phantom Type 是 Evidence 模式的一种推广。

一些关于 Evidence 模式的讨论文章：

[https://cs-syd.eu/posts/2016-07-24-overcoming-boolean-blindness-evidence.html](https://cs-syd.eu/posts/2016-07-24-overcoming-boolean-blindness-evidence.html)

[cs-syd.eu/posts/2016-07-24-overcoming-boolean-blindness-evidence.html](https://cs-syd.eu/posts/2016-07-24-overcoming-boolean-blindness-evidence.html)

[https://runtimeverification.com/blog/code-smell-boolean-blindness/](https://runtimeverification.com/blog/code-smell-boolean-blindness/)

[runtimeverification.com/blog/code-smell-boolean-blindness/](https://runtimeverification.com/blog/code-smell-boolean-blindness/)

[Parse, don’t validate](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/)

[lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/)

[https://kataskeue.com/gdp.pdf](https://kataskeue.com/gdp.pdf)