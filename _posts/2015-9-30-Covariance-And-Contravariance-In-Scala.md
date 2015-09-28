---
layout: post
title: Covariance and contravariance in Scala
---
This post explains concepts of covariance and contravariance. It focuses on real world metaphors and on code examples that show how those concepts can be used in Scala. Metaphors are borrowed from Erik Meijer's talk _Contravariance is the Dual of Covariance Implies Iterable is the Dual of Observable (Making Money Using Math)_ [Meijer2014].

## Covariance
<a name="covariance"></a>
First example of the covariance section uses model of drinks presented below.

![diagram]({{ site.baseurl }}/images/Covariance-And-Contravariance-In-Scala/drinks_model.png)

Imagine that your employer promised a new soft drink vending machine. Covariance means that in its place he can install a cola or tonic water vending machine because both cola and tonic water are subtypes of soft drink. Let's turn this description into Scala code.

{% highlight scala %}
class VendingMachine[+A] {
  // .. don't worry about implementation yet
}

def install(softDrinkVM: VendingMachine[SoftDrink]): Unit = {
  // Installs the soft drink vending machine
}
{% endhighlight %}

`install` method accepts a `VendingMachine` of type `SoftDrink` or subtypes of `SoftDrink` (`Cola` and `TonicWater`). This is possible because type parameter `A` is prefixed with a +. It indicates that subtyping is covariant in that parameter. Alternatively, it can be said that class `VendingMachine` is covariant in its type parameter `A`. Next code snippet shows covariant subtyping because `Cola` and `TonicWater` are subtypes of `SoftDrink`.

{% highlight scala %}
// covariant subtyping
install(new VendingMachine[Cola])
install(new VendingMachine[TonicWater])
// invariant
install(new VendingMachine[SoftDrink])
{% endhighlight %}

However, a `VendingMachine` of type `Drink` can't be passed to `install` method because `Drink` is a supertype of `SoftDrink`. That would be contravariant subtyping. [Contravariance is explained later](#contravariance).

{% highlight scala %}
// Compile error ! contravariant subtyping
install(new VendingMachine[Drink])
{% endhighlight %}

To sum up.

{% highlight scala %}
// Covariant subtyping
               A  <:                B
VendingMachine[A] <: VendingMachine[B]
{% endhighlight %}

If `A` is a subtype of `B` then `VendingMachine[A]` should be a subtype of `VendingMachine[B]`. This property is called covariant subtyping [Odersky2014].

### <a name="covariantuserestrictions"></a> Use restrictions of covariant type parameter

When type parameter is declared as covariant, the number of positions where it can be safely used is reduced. Thankfully, Scala compiler prevents from the use of covariant type parameter in positions that could lead to potential errors. Next example, this time from Java, shows why this is important. 

Unfortunately, arrays in Java are covariant. This allows code below to compile because `String` is a subtype of `Object`. Therefore, covariant subtyping can be applied.

{% highlight java %}
Object[] objects = new String[] { "abc" };
{% endhighlight %}

If `objects` will be modified with value of type other than `String` then `ArrayStoreException` will be thrown.

{% highlight java %}
String[] strings = { "abc" };
Object[] objects = strings;
// OOPS! Line below throws runtime exception: ArrayStoreException.
// Reason is that objects is actually an instance of 
// Array of Strings and we try to update it with an Integer.
objects[0] = 1;
{% endhighlight %}

Scala fixes this problem by making its [Array](http://www.scala-lang.org/api/current/#scala.Array) type invariant.

{% highlight scala %}
// Simplified Array from Scala
final class Array[T](_length: Int) {
  // arr.update(i, x) is equivalent to arr(i) = x
  def update(i: Int, x: T): Unit = ...
  // ...
}
{% endhighlight %}

Scala's `Array` type parameter doesn't have covariance annotation (+ prefix), if it had, the method `def update(i: Int, x: T): Unit` could lead to potential runtime errors. And if it actually did have a + prefix, Scala compiler would have reported compile error.

### <a name="legalcovariantpositions"></a> Legal positions of covariant type parameter

In general, covariant type parameter can be used as immutable field type, method return type and also as method argument type if method argument type has a lower bound. Because of those restrictions, covariance is most commonly used in producers (types that return something) and immutable types. Those rules are applied in the following implementation of `VendingMachine`.

{% highlight scala %}
class VendingMachine[+A](val currentItem: Option[A], items: List[A]) {

  def this(items: List[A]) = this(None, items)

  def dispenseNext(): VendingMachine[A] =
    items match {
      case Nil => {
        if (currentItem.isDefined)
          new VendingMachine(None, Nil)
        else
          this
      }
      case t :: ts => {
        new VendingMachine(Some(t), ts)
      }
    }

  def addAll[B >: A](newItems: List[B]): VendingMachine[B] =
    new VendingMachine(items ++ newItems)

}
{% endhighlight %}

The method `def addAll[B >: A](newItems: List[B]): VendingMachine[B]` has very useful characteristic. Lower bound makes `addAll` very flexible as seen below.

{% highlight scala %}
val colasVM: VendingMachine[Cola] = 
                    new VendingMachine(List(new Cola, new Cola))
val softDrinksVM: VendingMachine[SoftDrink] = 
                    colasVM.addAll(List(new TonicWater))
{% endhighlight %}

`SoftDrink` is the common supertype of both `TonicWater` and `Cola`, so `addAll` above returns `VendingMachine` of type `SoftDrink`. 

#### <a name="variancemutablefieldtype"></a> Covariant (and contravariant) type parameter as mutable field type

Type parameter with variance annotation (covariant + or contravariant -) can be used as mutable field type only if the field has object private scope (`private[this]`). This is explained in Programming In Scala [Odersky2008].

> Object private members can be accessed only from within the object in which they are defined. It turns out that accesses to variables from the same object in which they are defined do not cause problems with variance. The intuitive explanation is that, in order to construct a case where variance would lead to type errors, you need to have a reference to a containing object that has a statically weaker type than the type the object was defined with. For accesses to object private values, however, this is impossible.

Personally, I can't imagine there being many good use cases for using covariant type parameter as mutable field type. However, I have prepared an example that shows how rule above could be applied in practice. Imagine that you are creating a sci-fi shooter game with different kinds of bullets.

{% highlight scala %}
trait Bullet
class NormalBullet extends Bullet
class ExplosiveBullet extends Bullet
{% endhighlight %}

Bullets are contained in the the `AmmoMagazine` as seen in the next code listing. Notice that class `AmmoMagazine` is covariant in its type parameter. It also has mutable field `bullets` which compiles because of object private scope. Every time `giveNextBullet` is invoked, bullet from `bullets` list is removed. `AmmoMagazine` can't be refilled with bullets and there is no way of introducing this feature into this class because [that would lead to potential runtime errors](#covariantuserestrictions).

{% highlight scala %}
final class AmmoMagazine[+A <: Bullet](
    private[this] var bullets: List[A]) {
	
  def hasBullets: Boolean = !bullets.isEmpty

  def giveNextBullet(): Option[A] =
    bullets match {
      case Nil => {
        None
      }
      case t :: ts => {
        bullets = ts
        Some(t)
      }
    }
	
}
{% endhighlight %}

`AmmoMagazine` is used in the `Gun` class. `Gun` has method `reload` which thanks to covariant subtyping can be reloaded with both `AmmoMagazine[NormalBullet]` and `AmmoMagazine[ExplosiveBullet]`, and in general with `AmmoMagazine` of any subtype of `Bullet`.

{% highlight scala %}
final class Gun(private var ammoMag: AmmoMagazine[Bullet]) {

  def reload(ammoMag: AmmoMagazine[Bullet]): Unit =
    this.ammoMag = ammoMag  

  def hasAmmo: Boolean = ammoMag.hasBullets

  /** Returns Bullet that was shoot or None if there is ammo left */
  def shoot(): Option[Bullet] = ammoMag.giveNextBullet()

}
{% endhighlight %}

{% highlight scala %}
val gun = new Gun(AmmoMagazine.newNormalBulletsMag)
// compiles because of covariant subtyping
gun.reload(AmmoMagazine.newExplosiveBulletsMag)  
{% endhighlight %}

## Contravariance ##
<a name="contravariance"></a>

This section is very similar to the previous one because contravariance is simply the opposite of covariance. Example of contravariance section uses model of items presented below.

![diagram]({{ site.baseurl }}/images/Covariance-And-Contravariance-In-Scala/items_model.png)

Long time ago all kinds of trash could be put into single garbage can. Nowadays, trash must be segregated into different garbage cans - one for plastic items, one for paper items and so on. Contravariance means that if you need a garbage can for plastic items, you can simply set in its place a garbage can for items, because item is the supertype of plastic item. In Scala, this can be expressed as follows.

{% highlight scala %}
class GarbageCan[-A] {
  // .. don't worry about implementation yet
}

def setGarbageCanForPlastic(gc: GarbageCan[PlasticItem]): Unit = {
  // sets garbage can for PlasticItem items
}
{% endhighlight %}

`setGarbageCanForPlastic` method can accept a `GarbageCan` of type `PlasticItem` or supertype of `PlasticItem` (`Item`). This is possible because type parameter `A` is prefixed with a -. It indicates that subtyping is contravariant in that parameter. Alternatively, it can be said that class `GarbageCan` is contravariant in its type parameter `A`. Next code snippet shows contravariant subtyping because `Item` is the supertype of `PlasticItem`.

{% highlight scala %}
// contravariant subtyping
setGarbageCanForPlastic(new GarbageCan[Item])

// invariant
setGarbageCanForPlastic(new GarbageCan[PlasticItem])
{% endhighlight %}

However, a `GarbageCan` of type `PlasticBottle` can't be passed to `setGarbageCanForPlastic` method, because `PlasticBottle` is the subtype of `PlasticItem`. [That would be covariant subtyping](#covariance). 

{% highlight scala %}
// Compile error ! covariant subtyping
setGarbageCanForPlastic(new GarbageCan[PlasticBottle])
{% endhighlight %}

To sum up.

{% highlight scala %}
// Contravariant subtyping
           A  <:            B
GarbageCan[B] <: GarbageCan[A]
{% endhighlight %}

If `A` is a subtype of `B` then `GarbageCan[B]` should be a subtype of `GarbageCan[A]`. This property is called contravariant subtyping.

### Use cases for contravariant type parameter ###

Contravariant type parameter is usually used as method argument type, so naturally contravariance is most commonly seen in consumers (types that accept something). It can also be used as mutable field type if the field has object private scope as explained [before](#variancemutablefieldtype). Scala compiler prevents from the use of contravariant type parameter in positions that could lead to potential errors. If contravariant type parameter is used in illegal position such as method return type then Scala compiler reports an error. Typical use of contravariant type parameter is applied in the following implementation of `GarbageCan`.

{% highlight scala %}
class GarbageCan[-A] {

  // Instead of object private scope, 
  // lower bound can be used.
  // type B >: A
  // private var items: List[B] = _ 

  // compiles because of object private scope
  private[this] var items: List[A] = List.empty

  def put(item: A): Unit = this.items :+= item

  def putAll(items: List[A]): Unit = this.items ++= items

  def itemsCount: Int = this.items.size

}
{% endhighlight %}

## Real world use cases ##

Covariance and contravariance is actually quite common in functional style programming. This section shows example use cases from real APIs used by many developers on a daily basis.

### Function type (T) => R ###

One of the most common type used in the functional programming style is [function (T) => R](http://www.scala-lang.org/api/current/#scala.Function1). 

{% highlight scala %}
// a bit simplified source code from Scala API
trait Function1[-T, +R] extends AnyRef { 
  def apply(v1: T): R
}
{% endhighlight %}

In this type both contravariance and covariance is used. `T` is contravariant type parameter because it is used as `apply` input type and `R` is covariant type parameter because it is used as `apply` return type. This makes type `Function1` very flexible. Let's examine how contravariance in `Function1` helps to achieve more with less code. Below code snippet presents music instruments model.

{% highlight scala %}
trait MusicInstrument {
  val productionYear: Int
}
case class Guitar(val productionYear: Int) extends MusicInstrument   
case class Piano(val productionYear: Int) extends MusicInstrument
{% endhighlight %}

Next code snippet shows a function that returns true if MusicInstrument is vintage.

{% highlight scala %}
val isVintage: (MusicInstrument => Boolean) = _.productionYear < 1980
{% endhighlight %}

Because of covariant subtyping, this function can be used to [filter](http://www.scala-lang.org/api/current/index.html#scala.collection.immutable.List@filter(p:A=>Boolean):Repr) both `List[Piano]` and `List[Guitar]`. If `T` from function `(T) => R` was not a contravariant type parameter then two versions of `isVintage` function would have to be implemented, one for `Piano` and one for `Guitar`. See it in action code listing below.

{% highlight scala %}
val isVintage: (MusicInstrument => Boolean) = _.productionYear < 1980

test("should filter vintage guitars") {
  // given
  val guitars: List[Guitar] = List(new Guitar(1966), new Guitar(1988))
  // when
  val vintageGuitars: List[Guitar] = guitars.filter(isVintage)
  // then
  assert(List(new Guitar(1966)) === vintageGuitars)
}

test("should filter vintage pianos") {
  // given
  val pianos: List[Piano] = List(new Piano(1975), new Piano(1985))
  // when
  val vintagePianos: List[Piano] = pianos.filter(isVintage)
  // then
  assert(List(new Piano(1975)) === vintagePianos)
}
{% endhighlight %}

Example above only took advantage of contravariant type parameter of `(T1) => R` but covariant type parameter `R` can be just as useful.

### Immutable container types - Option, Collections ###

Scala immutable container types such as [`Option`](http://www.scala-lang.org/api/current/#scala.Option) and [`List`](http://www.scala-lang.org/api/current/#scala.collection.immutable.List) are covariant in its type parameter. If this wasn't the case then in the code snippet below, separate `getPrices` method for each subtype of `MusicInstrument` would have to be implemented. However, with the use of covariant subtyping, one method is enough.

{% highlight scala %}
def getPrices(instruments: List[MusicInstrument]) = {
  // returns prices of instruments    
}

val guitars: List[Guitar] = List(new Guitar(1966), new Guitar(1988))  
// covariant subtyping because Guitar is a subtype of MusicInstrument
val guitarsPrices = getPrices(guitars)
{% endhighlight %}

### RxScala ###

Asynchronous and event-based programs have been gaining a lot of popularity in the recent years. It used to be really challenging to write such programs but with the arrival of [Reactive Extensions](http://reactivex.io/) things have become much easier. Quote below explains what are Reactive Extensions in short [Reactivex.io].

> ReactiveX is a library for composing asynchronous and event-based programs by using observable sequences.
>
>It extends the observer pattern to support sequences of data and/or events and adds operators that allow you to compose sequences together declaratively while abstracting away concerns about things like low-level threading, synchronization, thread-safety, concurrent data structures, and non-blocking I/O.

Reactive extensions have been implemented in many languages, including Scala in project [RxScala](http://reactivex.io/rxscala/). What does it have to do with this post? This library makes heavy use of covariance and contravariance. Just look at two [`Observable`](http://reactivex.io/rxscala/scaladoc/index.html#rx.lang.scala.Observable) and [`Observer`](http://reactivex.io/rxscala/scaladoc/index.html#rx.lang.scala.Observer) which are two most important types from this library. Those types are examined next and a new advanced concept of flipped classification is introduced.

#### Flipped classification ####

{% highlight scala %}
trait Observable[+T] {  
  def subscribe(observer: Observer[T]): Subscription  
  def map[R](func: T => R): Observable[R]  
  // much more  
}
{% endhighlight %}

`Observable` is a type that produces values of type `T`, so it is covariant in type parameter `T`. However, if you look at method `subscribe` you might think that it uses type `T` in illegal position. After all, it was explained before that if type parameter is declared as covariant then it can be used as method argument type if it has a lower bound (`[B >: T]`). So the question is, why `subscribe` above compiles? The answer is flipped classification. Method `subscribe` accepts parametrized type `Observer` as an argument. The trick is that `Observer` is contravariant in its type parameter `T`.

{% highlight scala %}
trait Observer[-T] {
  def onNext(value: T): Unit  
  // .. more methods
}
{% endhighlight %}

Flipped classification also applies to `map` method of `Observable`. Method `map` accepts as an argument type `(T => R)`, that is `Function1[-T, +R]`. `Function1` is contravariant in type parameter T so flipped classification is also applied by the Scala compiler. 

Flipped classification is explained in more detail in in _The fast track_ section of Type Parameterization chapter in [Programming In Scala](http://www.artima.com/pins1ed/type-parameterization.html). 

## Use-site and declaration-site variance ##

So far this post has shown declaration-site variance because variance was defined during the declaration of the type by its creator, like below:

{% highlight scala %}
class VendingMachine[+A]
{% endhighlight %}

There is also use-site variance in which variance is defined by the user of the type. For example, see `VendingMachine` below which is invariant in its type parameter `A`.

{% highlight scala %}
class VendingMachine[A] {
  // Vending Machine is invariant in type parameter A, so
  // you can use that type parameter however you want.
}
{% endhighlight %}

If the user of the `VendingMachine` would like to use covariant subtyping then he would have to define covariance himself, for example like in the code below.

{% highlight scala %}
/**
 * Use-site covariance using bounds. Accepts a Vending Machine
 * of type SoftDrink or subtypes of SoftDrink (Cola or TonicWater).
 */
def install(softDrinkVM: VendingMachine[_ <: SoftDrink]): Unit = {
  // Installs soft drink vending machine
}
{% endhighlight %}

`<:` is like `extends` from Java. It looks pretty ugly doesn't it? If `install` method was defined as `install(softDrinkVM: VendingMachine[SoftDrink])` then code above wouldn't compile because it would require the types to match exactly. That is, only `install(new VendingMachine[SoftDrink])` would have worked.

So besides this being pretty ugly, there is also another problem. Take a look at class `Box` defined in the next code listing. `Box` uses type parameter `A` as both output and input, so it can't be declared by the creator of the class as neither covariant nor contravariant. 

{% highlight scala %}
class Box[A]() {
  private var _thing: A = _

  def retrieve: A = _thing

   // explicit setter for the sake of example    
   def put(thing: A) = this._thing = thing
}
{% endhighlight %}

Imagine user wants to both put and retrieve items from the Box.

{% highlight scala %}
val box: Box[SoftDrink] = new Box()
box.put(new Cola)
val thing: SoftDrink = box.retrieve
{% endhighlight %}

If the user would like covariant subtyping to work with the class `Box` then he would have to add variance himself. However, this will cause method `put` to be no longer usable, because compiler will not be able to figure out the correct type for the parameter of `put` method.

{% highlight scala %}
val box: Box[_ <: SoftDrink] = new Box()
// type mismatch; found : Cola required: _$4
// box.put(new Cola)
val thing: SoftDrink = box.retrieve
{% endhighlight %}

Similar problem exists when user wants to use contravariant subtyping.  

{% highlight scala %}
val box: Box[_ >: SoftDrink] = new Box()
box.put(new Cola)
// type mismatch; found : _$2 required: SoftDrink    
// val thing: SoftDrink = box.retrieve
{% endhighlight %}

Covariance and contravariance can be quite tricky to get right so it is best to define them during the declaration of the type. And finally, it also leads to really ugly and hard to read API. Java only has use-site variance and it doesn't make its API look pretty.... tadada

## Summary ##

Covariance and contravariance are something that many take for granted. It is especially relevant now because of the functional style programming that has gained a lot of popularity in the recent years. If covariance and contravariance would be suddenly turned off, a lot of code wouldn't compile any more.

It is important that both library developers and users understand concepts of covariance and contravariance. Covariance and contravariance makes libraries more generic and lets their aware users achieve more functionality with less code. 

From the perspective of library developer it is easiest to remember that covariant type parameter should be most often used as output type and contravariant type parameter as input type. If you want to use type parameter as both input and output type, then it should be invariant.

From the perspective of library user it is best to remember rules below. First covariant subtyping.

{% highlight scala %}
// Covariant subtyping (Vending machine metaphore)
               A  <:                B
VendingMachine[A] <: VendingMachine[B]
{% endhighlight %}

And next contravariant subtyping which is the opposite of covariant subtyping.

{% highlight scala %}
// Contravariant subtyping (Garbage can metaphore)
               A  <:                B
VendingMachine[B] <: VendingMachine[A]
{% endhighlight %}

If you forget rules above, try to remind yourself of vending machine and garbage can metaphors. Last of all, all code examples (and more) are available at [github](https://github.com/kamkor/covariance-and-contravariance-examples).

## References ##

[Meijer2014] Meijer, Erik. _Contravariance is the Dual of Covariance Implies Iterable is the Dual of Observable(Making Money Using Math)_, 2014.<br/><<https://vimeo.com/98922027>>

[Odersky2014] Odersky, Martin. _Scala By Example_, pages 56-58. 2014.<br/><<http://www.scala-lang.org/docu/files/ScalaByExample.pdf>>

[Odersky2008] Odersky, M., Parrow, J., Walker, D.: _Programming in Scala: A comprehensive Step-by-step Guide_. Artima Incorporation, 2008.<br/><<http://www.artima.com/pins1ed/type-parameterization.html>>

[Bloch2008] Bloch, Joshua. _Effective Java: Second Edition_. Addison-Wesley, Boston, 2008

[Reactivex.io] <http://reactivex.io/intro.html>

## Terms table ##

There are a lot of terms connected with covariance, contravariance and generics in general. Table below summarizes these terms. It contains a lot of terms and examples from [Bloch2008] and some new ones related to variance annotations. 

Term  | Scala Example | Java Example
-------------------------------- | -------------------------------- | --------------------------------
Parametrized type                | ```List[String]```               | ```List<String>```
Actual type parameter            | ```String```                     | ```String```
Generic type                     | ```List<A>```                    | ```List<E>```
Formal type parameter            | ```A```                          | ```E```
Unbounded wildcard type          | ```List[_]```                    | ```List<?>```
Raw type             | ```List```             | ```List```
Type parameter with lower bound  | ```[A >: Number]```              | ```<E super Number>```
Type parameter with upper bound  | ```[A <: Number]```                    | ```<E extends Number>```
Wildcard type with lower bound   | ```[_ >: Number]```          | ```<? super Number>```
Wildcard type with upper bound   | ```[_ <: Number]```          | ```<? extends Number>```
Recursive type bound       | ```[A <: Ordered[A]]```        | ```<T extends Comparable<T>>```
Type constructor         | ```List, constructs List[Int] etc```   | ```Same as in Scala```
Variance annotation              | ```+ or - i.e. [+A] or [-A]```          | ```not supported```
Covariance annotation            | ```+ i.e. [+A]```                       | ```not supported```
Contravariance annotation        | ```- i.e. [-A]```                       | ```not supported```

