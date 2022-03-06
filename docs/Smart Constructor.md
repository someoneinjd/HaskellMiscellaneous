上回 （NewType） 说到用 haskell 编写一个扫雷，并且使用 `Position` 类型表达每个格子在雷场中的位置。

好像忘了什么? 对了，应该用 9×9 的雷场。也就是说 `Position (i, j)` 应该满足 `(0 <= i < 9, 0 <= j < 9)`

你看到了，haskell 的常规类型检查对于这种「细分」的新类型束手无策 （Dependent Type 可以解决，但是 haskell 目前还没有……），所以我们需要一些额外的设计，SmartConstructor正是这样的一种模式。

其实非常简单，用一个新的函数充当类型构造子，并且加上范围检查就是了。

```haskell
mkPos :: Int -> Int -> Position
mkPos x y | x < 9 && x >= 0 && y < 9 && y >= 0 = Position (x, y)
          | otherwise = error "MineSweeper.Position.mkPos : 不合法的参数范围"
-- 错误报告简化了
```

你可能会问如何避免有人绕过 `mkPos` 直接使用 `Position`，答案是定义 module 时将 `Position `作为内部函数隐藏起来，只对外部提供 `mkPos`。如果你不喜欢 `Error`，也可以用 `Maybe` 等 Monad 表达错误。

注：隐藏 `Position` 这种直接值构造子的目的还有一个，防止对 `Data.Coerce` 的误用。

但是隐藏了值构造子，也就不能用 Pattern Matching 了，如果使用的话，编译时GHC 会给出这样的错误：

```
error: Not in scope: data constructor ‘Position
```

所以还得弄点方便访问的辅助函数。总结全代码如下

```haskell
module MineSweeper.Position (mkPos, fstP, sndP, unPos) where

newtype Position = Position (Int,Int)
mkPos :: Int -> Int -> Position
mkPos x y | x < 9 && x >= 0 && y < 9 && y >= 0 = Position (x, y)
          | otherwise = error "MineSweeper.Position.mkPos : 不合法的参数范围"

unPos :: Position -> (Int, Int)
unPos (Position i) = i

fstP, sndP :: Position -> Int
fstP = fst . unPos
sndP = snd . unPos
```

很简明，`unPos` 提取整个二元组，`fstP`，`sndP` 的作用更加明了：无非是 `fst` 和 `snd` 的特化版本。

这好像已经是一个很健壮，很 Robust 的设计了，但是我们还有几个地方可供讨论。

1. 有无办法把参数性质检查放到编译期？

仔细想想，如果参数都是常量，那么在编译期提前执行自定义的构造子，既可以加速程序运行速度，又可以把一些运行时错误提早到编译期，这是符合haskell理念的(不过如果使用 Monadic 的错误处理，显然只能减少一些运算)

额，haskell 当然是提供了编译期运算的语言 设施的 --- 但是是GHC扩展。

Template Haskell

好的，现在来重写一下 `Position` 模块。

```haskell
{-# LANGUAGE DeriveLift #-}

module MineSweeper.Position (mkPos, fstP, sndP, unPos) where

import Language.Haskell.TH.Syntax

newtype Position = Position (Int,Int) deriving Lift

mkPos :: Int -> Int -> Position
mkPos x y | x < 9 && x >= 0 && y < 9 && y >= 0 = Position (x, y)
          | otherwise = error "MineSweeper.Position.mkPos : 不合法的参数范围"

unPos :: Position -> (Int, Int)
unPos (Position i) = i

fstP, sndP :: Position -> Int
fstP = fst . unPos
sndP = snd . unPos
```

我们做了啥？好像关键的函数什么也没有变啊！实际上就是没变，这里只是引入Lift  class 和为 `Position` 类型实现了 `Lift` 的实例而已。(import 的库包含了 `Lift`， 最顶端的 `{-# LANGUAGE  DeriveLift #-}` 开启一个扩展，使得 GHC 可以自动为 `Position` 类型生成 `Lift` 的实例)

之后呢？

此处提供 test.hs 如下

```haskell
{-# LANGUAGE TemplateHaskell #-}

import MineSweeper.Position
import Language.Haskell.TH.Syntax


main = print $(lift (mkPos 9 9))
```

现在该说一些更细节的东西了，TemplateHaskell 是一个为 haskell 实现模板元编程的扩展，但是实际上还包括了 template-haskell 库。它主要的作用就是在编译期执行运算生成 AST。

lift Expr 会把 Expr 当成一个 haskell 表达式求值，如果它确实可以在编译期计算，就把结果转换成对应的 AST 并替换到 （lift ..） 所 在的位置。

太长不看版 ：想在编译期就搞定某些计算就用 `lift`，同时括号内的函数必须来自其他分隔开的 Module。

对了，直接编译 test.hs，GHC 会提醒你：

```
ghc: this operation requires -fexternal-interpreter
```

编译期运算是通过外部解释器实现的，加上 `-fexternal-interpreter` 才能真正完成编译!

更多细节：[Using Template Haskell to generate static data](https://well-typed.com/blog/2020/06/th-for-static-data/)

更更多的细节：[Language.Haskell.TH.Syntax](https://hackage.haskell.org/package/template-haskell-2.16.0.0/docs/Language-Haskell-TH-Syntax.html)

不要一头热的觉得你找到了某种银弹，看看这里，谨慎使用TemplateHaskell：[What's so bad about Template Haskell?](https://stackoverflow.com/questions/10857030/whats-so-bad-about-template-haskell)

其实实现编译期运算也不止 TH 一种方式：[https://dev.to/serokell/compile-time-evaluation-in-haskell-58j4](https://dev.to/serokell/compile-time-evaluation-in-haskell-58j4)

建议看个乐，别太当真了。

2. GHC 有个扩展叫 `PatternSynonyms`，如果把原本的构造子隐藏起来，那很抱歉这个看起来就用不了了。(我没看具体的实现原理，只是做 了个测试，有谁来个全面挖掘吗？)

(看看这里了解一下基本上它用来做什么：[Pattern Synonyms](https://kseo.github.io/posts/2016-12-22-pattern-synonyms.html)

3. Monadic 的错误处理考虑一下 `Either String` 和 `Validation`。