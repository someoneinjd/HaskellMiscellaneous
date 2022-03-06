## Collecting all Images

想象一下你正在用 haskell 开发一个 GUI 下的游戏（估计我一辈子都不会用 haskell 写出啥有用的程序了）需要确认各个场景的对应图像文件都在。

首先把场景（`Scene`）的类型设计好。

```haskell
data Scene = Scene
  { backgroundImage   :: Text
  , characters        :: [Character]
  , bewilderedTourist :: Maybe Character
  , objects           :: [Either Rock WoodenCrate]
  }

data Character = Character
 { hat   :: Maybe DamageArray
 , head  :: DamageArray
 , torso :: DamageArray
 , legs  :: DamageArray
 , shoes :: Maybe DamageArray
 }

data DamageArray = DamageArray
  { noDamage        :: Text
  , someDamage      :: Text
  , excessiveDamage :: Text
  }

data Rock = Rock
  { weight    :: Double
  , rockImage :: Text
  }

data WoodenCrate = WoodenCrate
  { strength         :: Double
  , woodenCrateImage :: DamageArray
  }
```

要做的事情就是把对应图像文件名的 `Text` 收集起来，去重，挨个检查。最适合做这个容器的显然是 `Data.Set`

```haskell
collectImages :: Scene -> Set Text
```

函数签名设计完毕！

但是我们可以看到，除了 `Scene` 本身所存储的图像文件名，还有很多图像文件名的信息是放在 `Scene` 所存储的子结构里的。甚至还带有 `Maybe`，`Either` 等上下文。

也许比较合适的方式是写多个函数，把每个子结构都考虑到。

```haskell
collectImages :: Scene -> Set Text
collectImages Scene {..}
  =  singleton backgroundImage
  <> mconcat (map collectCharacterImages characters)
  <> maybe mempty collectCharacterImages bewilderedTourist
  <> mconcat (map (either (singleton . collectRockImage)
                          collectWoodenCrateImages)
                  objects)
-- {..}是 RecordWildCards 扩展提供的语法糖
-- 食用方法: https://ocharles.org.uk/blog/posts/2014-12-04-record-wildcards.html
-- 其他几个函数就不写了，多半对你很简单
```

一堆函数……起名应该是程序员的第一难题了吧。是时候用 typeclass 来把这一堆收拾一下了。先写出这些函数的类型签名

```haskell
collectImages :: Scene -> Set Text
collectCharacterImages :: Character -> Set Text
collectDamageArrayImages :: DamageArray -> Set Text
collectRockImage :: Rock -> Text
collectWoodenCrateImages :: WoodenCrate -> Set Text
```

这么一看，其实它们的类型签名服从一个共同的模式：`a -> Set Text`，但有一个是 `Rock -> Text`，为了一致性，我们包裹一下它的返回值，改成 `Set Text` 类型。这会带来性能上的损失，但是也有不少优点。

那么相应的 typeclass 设计如下：

```haskell
class HasImages a where
  images :: a -> Set Text
```

那么应该开始着手给 `Scene`， `Character`，`DamageArray` 几个写实例了吗？先不急， 我们先定义几个派生规则，搞定`Maybe`，`Either`，`List` 这几个「上下文」的处理。

把上下文处理单独拉出来写成实例是有好处的，如果在未来，`Scene` 类型的定义需要修改，又加入了一些新的内容，并且仍然使用了这几个上下文的其中之一，这很有可能，那么就可以复用这几个实例的派生规则，好处多多。

```haskell
instance HasImages a => HasImages [a] where
  images xs = foldr (\x accum -> images x <> accum) mempty xs

instance HasImages a => HasImages (Maybe a) where
  images x = maybe [] images x

instance (HasImages a, HasImages b) => HasImages (Either a b) where
  images x = either images images x

-- 然后
instance HasImages Scene where
  images Scene {..}
    =  singleton backgroundImage
    <> images characters
    <> images bewilderedTourist
    <> images objects

instance HasImages Character where
  images Character {..}
    =  images hat
    <> images head
    <> images torso
    <> images legs
    <> images shoes

instance HasImages DamageArray where
  images DamageArray {..} = fromList
    [ noDamage
    , someDamage
    , excessiveDamage
    ]

instance HasImages Rock where
  images Rock {..} = singleton rockImage

instance HasImages WoodenCrate where
  images WoodenCrate {..} = images woodenCrateImage
```

大功告成！虽然增加了一些语法噪音和缩进问题，但是总的来说，代码更方便使用了（不过，有点不好理解了）。

（可能有写过 Scheme 的朋友开始注意到了：怎么看着这么像个元求值器？虽然 `Scene` 类型并非一个 AST，但是就是有这种感觉！）

如果有一组类型 {a, b, c, d, e....}，它们都可以提取出类型为 μ 的值，就可以采用 HasIt 模式了。下面的例子进一步提供了 HasIt 模式的一个更加泛化、通用的例子。

## Convenient Argument Passing

假设要用 haskell 开发一个数据库应用，存点用户数据之类的东西。

```haskell
newtype Key a = Key UUID
data Entity a = Entity
  { entityKey   :: Key a
  , entityValue :: a
  }
```

UUID 是通用唯一识别码 （Universally Unique Identifier）的缩写，总之就是用户在你这个平台的数据身份证，每个用户的 UUID 都是唯一的。

现在要写个函数针对某个用户查询的好友，是账户的好友不是朋友。

```haskell
getFriends :: Key User -> [Entity User]
```

你可能会有点疑惑，`Key a` 中的类型变量 `a` 和值其实没半点瓜葛，为什么不像下面这样定义呢？

```haskell
newtype Key = Key UUID
```

这实际上是一种被称为 “PhantomType” 的设计模式，在类型构造子的参数处刻意多出一或多个根本和值无关的参数，这些额外的类型参数可以自由地用于传递一些信息。当然了，一般来说，如果需要在类型中传递的信息或者信息处理的逻辑较为复杂，那就用 GADTs 和其他一些扩展。

回到正题，我们可能经常需要从 `Entity User` 去 unpack 出 `Key User` 使用，比如找出一个用户的好友的好友

```haskell
concatMap (getFriends . entityKey) (getFriends user)
```

那就不如弄个好用一点的 API

```haskell
class HasKey a k | a -> k where
  key :: a -> Key k
```

这里使用的扩展为：`MultiParamTypeClasses`、`FunctionalDependencies`，前者开启多参数的 typeclass，后者则是那个 ` | a ->  k` 的来源，意思是由参数 `a` 可以唯一确定一个对应的 `k`，所以叫类似函数的依赖机制 。由类型 `a` 可以确定唯一的类型 `k` ，听起来就很像函数嘛。

一个简单的使用教程在此： [24 Days of GHC Extensions: Functional Dependencies](https://ocharles.org.uk/blog/posts/2014-12-14-functional-dependencies.html)

一篇复杂的博文在此： [https://aphyr.com/posts/342-typing-the-technical-interview](https://aphyr.com/posts/342-typing-the-technical-interview)

最后实现 `HasKey` 的实例和 `getFriends`

```haskell
instance HasKey (Key a) a where
  key = id

instance HasKey (Entity a) a where
  key = entityKey

getFriends :: HasKey a User => a -> [Entity User]
```

此外，HasIt 模式很适合结合 haskell 的 MTL 库使用。这里不再赘述。在下面这篇文章中，你会看到一开始的简单模式在结合上不动点理论、F-Algebra 和 FreeMonad 之后可以做些什么。

[http://www.cs.ru.nl/~W.Swierstra/Publications/DataTypesALaCarte.pdf](http://www.cs.ru.nl/~W.Swierstra/Publications/DataTypesALaCarte.pdf)