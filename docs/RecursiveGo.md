```haskell
-- 米娜桑应该也知道 haskell 的这个语法糖
--  [1..10]
-- 这样可以生成1~10的列表
-- 但是反过来想，怎么生成10~1的列表呢？
-- 看来有必要自己写函数了，如此简单的功能，想必对任何haskeller都是小菜一碟

range :: Int -> Int -> [Int]
range from to | from == to = [to]
              | from < to = [from..to]
              | from > to = go from where
                   go i | i == to = [i]
                        | otherwise = i : (go (i - 1))
```

这里就有个问题了，为什么不直接让 `range` 直接递归？ 原因之一是节省运行成本，毕竟外层的 guard 在逻辑上其实只要执行一次就好了，次数多了会拖慢执行速度。原因之二是，作为参数的 `to` 实际上根本不需要变化，如果仍然把它留在递归调用中，实在是很不清爽。

这种时候的一般做法就是用 `where` 定义一个叫 `go` 的函数，用它完成递归。scheme 语言中也有实现类似机制的语法糖，叫做 `let loop`。

至于为什么叫 `go`，我也不清楚是什么时候的习惯了，但是我最早看到 `go` 出现的文献是这一篇：[https://www.microsoft.com/en-us/research/wp-content/uploads/2001/09/rules.pdf](https://www.microsoft.com/en-us/research/wp-content/uploads/2001/09/rules.pdf)

2001年……haskell98 的上古圣遗物了属于是

(* 吐槽一下，我感觉这个 Pattern 的作用其实就是解决了一个常见场景的命名问题，外加可以理直气壮地说我没有在命名上偷懒，我这是顺应社区传统..... *)

`range` 函数工作得很好，但是还有一些小🤏细节需要我们去注意，让我们把类型签名去掉再看看。

直接作用于 `from` 和 `to` 的函数有这几个：

```haskell
(==) (>) (<) (-)
```

`==` 来自 `Eq class`，`>`、`<`来自 `Ord class`， 都是 Generic 的函数

`(-)` 好像只能归纳到 `Num class` 上，但是注意一个☝️小细节，这里的例子是 -1，所以其实对于 `Int` 而言，它也可以替换成 `Enum class` 下的 `prev` 函数。

那么很清楚了，`range` 可以实现成一个 Generic 的函数。

```haskell
{-# LANGUAGE ScopedTypeVariables #-}

module Range where

rangeG :: forall a . (Enum a, Ord a, Eq a) => a -> a -> [a]
rangeG from to | from == to = [to]
               | from < to = [from..to]
               | from > to = go from where
                   go :: a -> [a]
                   go i | i == to = [i]
                        | otherwise = i : (go (pred i))
```

到此为止了吗？没有，让我们回想一下 `Enum` 的含义：枚举。那么其实对正常编程中会出现的符合 `Enum class` 要求的类型，做一个到 `Int` 的转换，很简单。

[https://www.haskell.org/onlinereport/basic.html](https://www.haskell.org/onlinereport/basic.html)

一番查询后找到了 `fromEnum :: a -> Int`。这下可以更加泛化实现了。

```haskell
rangeG :: forall a . Enum a => a -> a -> [a]
-- 现在的约束条件不可谓不简洁, 但是分支上过分 繁冗了，稍作修剪。
rangeG from to | (fromEnum from) <= (fromEnum to) = [from..to]
               | (fromEnum from) > (fromEnum to) = go from where
                   go :: a -> [a]
                   go i | (fromEnum i) == (fromEnum to) = [i]
                        | otherwise = i : (go (pred i))
```

但是如果你真的看了上面的参考链接，你就会发现这个玩意:

```haskell
enumFromThenTo :: a -> a -> a -> [a] 
-- The sequence enumFromThenTo e1 e2 e3 is the list [e1,e1+i,e1+2i,...e3]
-- where the increment i is e2-e1.
-- (enumFromThenTo e1 e2 e3) == [e1,e2..e3]
```

白造轮子了属于是。标准库的实现更加泛化，甚至可以控制序列递进的步长，而且还有语法糖支持。

但是，仍然有些东西值得一提：

```haskell
{-# LANGUAGE ScopedTypeVariables #-}
```

这个扩展干嘛用的？

如果把上述示例代码中的 `go` 的类型签名去掉，代码仍然可以通过编译，而且其语义与之前一般无二，所以看起来在这里用扩展是自找麻烦(?)。

也许在一些更复杂的递归中，它能增加可读性吧，现在专注于扩展本身，这个扩展是做什么用的？（我不演了，我就是要 bb，就是要整类型体操）

在上面的示例代码中，`go` 的类型和 `range` 藕断丝连， 当 `range` 的类型变量落实到某个具体的类型时，`go` 里面的类型变量也要落实为相同的那一个类型。如果 `go `的变量约束和 `range `写成一样的，那么约束关系就是**平行**的，没法体现它们之间确实存在的关联，很不准确，而且对于有些复杂的例子，这样没办法让代码通过编译。

所以 `ScopedTypeVariable` 诞生了，一旦使用这个扩展，类型变量就可以像普通的函数变量一样，从上层作用域捕获。如果想要避免同名类型变量的捕获，需要手动加上 `forall` 才行，没有用 `forall` 声明的变量，全部会去上层作用域寻找，如果是 Toplevel 函数的话，就会因为找不到可以捕获的类型变量而无法通过类型检查。

让我们举一个原文中的题目作为例子：

```haskell
{-# LANGUAGE BangPatterns        #-}
{-# LANGUAGE ScopedTypeVariables #-}

sumGo :: forall a . Num a => [a] -> a
sumGo = go 0
  where
    go :: a -> [a] -> a
    go !acc [] = acc
    go acc (x:xs) = go (acc + x) xs
```

`sumGo` 中的类型变量 `a` 有 `forall` 声明，所以它是一个“fresh” 的变量，而 `go` 中的 `a` 没有标记，所以编译时会直接从 `sumGo` 的上下文中捕获具体的 `a` 所对应的类型。

可以一试的练手题：[Yoneda lemma](https://www.codewars.com/kata/5af33bcdde4c7f94a90000b3/haskell)

同时还有个问题，对于数运算的递归，一般来说总是习惯于使用尾递归，这样所花费的空间就和循环一样 了。为什么呢？大概是因为很多数类型的加法，乘法符合交换律，结合律（浮点数就不能这样草率了）。Scheme 代码中能找到很多这样的例子。但是对于 haskell ，还有一点细节问题要处理：haskell 的惰性求值策略，经典例子就是 `foldl` 实现为尾递归，却比 `foldr` 消耗更多空间。鉴于网上已经有很多讲解这部分的教程了，我也不制造类似的废话了.....关键词是 `WeakHeadNormalForm`，`seq` 和 `Data.DeepSeq`。

下面是名人名言水篇幅时间：

> 1976年，又要说回到苏联了，我因为吃生鱼片得了 严重的食物中毒。当住在医院的时候，在精神错乱的状态下，我忽然意识到，并行的加法计算能力依赖于加法  具有结合性的事实。（因此，简单地说，STL是细菌感染的结果。）换句话说，我意识到并行的减法运算是和半群结构类型有关联的。这就是基本点：算法是定义于代数结构基础之上的。意识到必须对正规公理加入复杂性必要条件以扩展结构的概念，又花了我另外一些年头，接着我又花了15年使之行得通。（我直到现在都 不能确定我已经成功地使我朋友小圈  子之外的任何人理解了这一点。）我相信迭代器理论是计算科学的中心，就象环或 Banach 空间理论是数学的中心一样。每当我找到一个算法时，我都要努力去寻求它所定义的结构基础。我想做的就是泛化地描述算法，并乐此不疲。我愿意花一个月时间去发现某个著名算法的泛化表示。 迄今为止，在向人们解释我 这种行为的重要性方面，我是异乎寻常的失败。但是，不知何故，这种行为的结果 ─ STL，却是如此成功。
>
> --  Alexander Stepanov

他和另一人合著的书已经变成自由书籍了，你可以在这里下载：[Elements of Programming](http://elementsofprogramming.com/)

非常邪道，非常代数。比大多数 haskell 书更重视代数，据此我觉得是时候把 haskell 啊 scheme 啊过分学术化的帽子拿掉了，C++才是学究气过分地强的语言啊。