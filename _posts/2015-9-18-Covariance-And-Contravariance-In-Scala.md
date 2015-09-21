---
layout: post
title: Covariance and contravariance in Scala
---
This post explains concepts of covariance and contravariance. It focuses on real world metaphors and on code examples that show how those concepts can be used in Scala. Metaphors are borrowed from Erik Meijer talk _Contravariance is the Dual of Covariance Implies Iterable is the Dual of Observable (Making Money Using Math)_ [Meijer2014].

## Covariance
<a name="covariance"></a>
First example of the covariance section uses model of drinks presented below:

![diagram]({{ site.baseurl }}/images/Covariance-And-Contravariance-In-Scala/drinks_model.png)

Imagine that your employer promised a new soft drink vending machine. Covariance means that in its place he can install a cola or tonic water vending machine, because both cola and tonic water are subtypes of soft drink. Let's turn this description into Scala code.

{% highlight scala %}
class VendingMachine[+A] {
  // .. don't worry about implementation yet
}

def install(softDrinkVM: VendingMachine[SoftDrink]): Unit = {
  // Installs the soft drink vending machine
}
{% endhighlight %}

`install` method can accept a `VendingMachine` of type `SoftDrink` or subtypes of `SoftDrink` (`Cola` or `TonicWater`). This is possible, because type parameter A is prefixed with a +. It indicates that subtyping is covariant in that parameter. Alternatively, it can be said that class `VendingMachine` is covariant in its type parameter `A`. Next code snippet shows covariant subtyping because `Cola` and `TonicWater` are subtypes of `SoftDrink`.

{% highlight scala %}
// covariant subtyping
install(new VendingMachine[Cola])
install(new VendingMachine[TonicWater])
// invariant
install(new VendingMachine[SoftDrink])
{% endhighlight %}

However, a `VendingMachine` of type `Drink` can't be passed to install method, because `Drink` is a super type of `SoftDrink`. That would be contravariant subtyping. [Contravariance is explained later in this post](#contravariance).

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

When a type parameter is declared as covariant, the number of places where it can be safely used is reduced (method argument type, field type etc.). Thankfully, Scala compiler prevents from the use of covariant type parameter in positions that could lead to potential errors. To understand why this is important, let's take a look at example from Java. Arrays in Java are covariant which means that code below compiles because `String` is a subtype of `Object`.

{% highlight java %}
Object[] arr = new String[3];
{% endhighlight %}

This leads to potential runtime errors.

{% highlight java %}
String[] strings = { "abc" };
Object[] objects = strings;
// OOPS! Line below throws the runtime exception: ArrayStoreException.
// Reason is, that objects is actually an instance of Array of Strings, and
// we try to update it with an Integer object.
objects[0] = 1;
{% endhighlight %}

[Scala fixes this by making its Arrays invariant](http://www.scala-lang.org/api/current/#scala.Array). Scala Array type parameter doesn't have covariance annotation (+ prefix), if it had, the method `def update(i: Int, x: T): Unit` could lead to potential runtime errors. And if it actually did have a + prefix, Scala compiler would have reported a compile error for `def update(i: Int, x: T): Unit` method declaration. So don't be afraid of using covariance because Scala compiler keeps you safe.

### <a name="legalcovariantpositions"></a> Legal positions of covariant type parameter ###

In general, covariant type parameter can be used for immutable field type, method return type and also for method argument type if method argument type has a lower bound. Because of those restrictions, covariance is most commonly used in producers (types that return something) and immutable types. Those rules are applied in the example [implementation of the `Vending Machine`](https://github.com/kamkor/covariance-and-contravariance-examples/blob/master/src/main/scala/kamkor/covariance/vendingmachine/VendingMachine.scala). Code snippet below shows `VendingMachine` trait. 

{% highlight scala %}
/** VendingMachine that is covariant in its type parameter. */
trait VendingMachine[+A] {

  /** Item currently in the slot. */
  def currentItem: Option[A]

  /**
   *  Returns a vending machine with next item in the currentItem or
   *  with none if there are no items left.
   */
  def dispenseNext(): VendingMachine[A]

  /** Returns a vending machine with added items. */
  def addAll[B >: A](newItems: List[B]): VendingMachine[B]  
  
}
{% endhighlight %}

The method `def addAll[B >: A](newItems: List[B]): VendingMachine[B]` has one more interesting characteristic. Because of the lower bound, it is legal to write

{% highlight scala %}
val colasVM: VendingMachine[Cola] = 
                    VendingMachine(List(new Cola, new Cola))
val softDrinksVM: VendingMachine[SoftDrink] = 
                    colasVM.addAll(List(new TonicWater))
{% endhighlight %}

`SoftDrink` is the common super type of both `TonicWater` and `Cola`, so `addAll` above returns a `VendingMachine` of type `SoftDrink`.

#### Covariant type parameter as mutable field type

Covariant type parameter can be used for mutable field type only if the field has object private scope (`private[this]`). This is explained in Programming In Scala [Odersky2008].

> Object private members can be accessed only from within the object in which they are defined. It turns out that accesses to variables from the same object in which they are defined do not cause problems with variance. The intuitive explanation is that, in order to construct a case where variance would lead to type errors, you need to have a reference to a containing object that has a statically weaker type than the type the object was defined with. For accesses to object private values, however, this is impossible.

Personally, I can't imagine there being many good use cases for using covariant type parameter for mutable field type. However, I have prepared an example that shows how rule above can be applied in Scala. Imagine that you are creating a sci-fi shooter game with different kinds of bullets.

{% highlight scala %}
trait Bullet
class NormalBullet extends Bullet
class ExplosiveBullet extends Bullet
{% endhighlight %}

Bullets are contained in the the ammo magazine as seen in the next code listing. Notice that class `AmmoMagazine` is covariant in its type parameter. It also has mutable field `bullets` which compiles because of object private scope. Every time `nextBullet` is invoked, bullet from bullets list is removed. Once ammo magazine is out of bullets, it can't refilled and there is no way of introducing this feature into this class because [that would lead to potential runtime errors](#covariantuserestrictions).

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

So where is the covariance used? In the `Gun` class. A `Gun` can be reloaded with `AmmoMagazine` of type `Bullet` which means that both `AmmoMagazine[NormalBullet]` and `AmmoMagazine[ExplosiveBullet]` can be passed to this method, and in general `AmmoMagazine` of any subtype of `Bullet`.

{% highlight scala %}
final class Gun(private var ammoMag: AmmoMagazine[Bullet]) {

  def reload(ammoMag: AmmoMagazine[Bullet]): Unit = {
    this.ammoMag = ammoMag  
  }

  def hasAmmo: Boolean = ammoMag.hasBullets

  /** Returns Bullet that was shoot or None if there is ammo left */
  def shoot(): Option[Bullet] = ammoMag.giveNextBullet()

}
{% endhighlight %}

{% highlight scala %}
val gun = new Gun(AmmoMagazine.newNormalBulletsMag)
// compiles, because of covariant subtyping
gun.reload(AmmoMagazine.newExplosiveBulletsMag)  
{% endhighlight %}

More details about how legal positions of covariant type parameter are determined can be found in _The fast track_ section of Type Parameterization chapter in [Programming In Scala](http://www.artima.com/pins1ed/type-parameterization.html).

## Contravariance ##
<a name="contravariance"></a>

This section is very similar to the previous one because contravariance is simply the opposite of covariance. Example of contravariance section uses model of items presented below.

![diagram]({{ site.baseurl }}/images/Covariance-And-Contravariance-In-Scala/items_model.png)

Long time ago all kinds of trash could be put into single garbage can. Nowadays, trash must be segregated into different garbage cans - one for plastic items, one for paper items and so on. Contravariance means that if you need a garbage can for plastic items, you can simply set in its place a garbage can for items, because item is the super type of plastic item. In Scala, this can be expressed as follows.

{% highlight scala %}
class GarbageCan[-A] {
  // .. don't worry about implementation yet
}

def setGarbageCanForPlastic(gc: GarbageCan[PlasticItem]): Unit = {
  // sets garbage can for PlasticItem items
}
{% endhighlight %}

`setGarbageCanForPlastic` method can accept a `GarbageCan` of type `PlasticItem` or super type of `PlasticItem` (`Item`). This is possible because type parameter A is prefixed with a -. It indicates that subtyping is contravariant in that parameter. Alternatively, it can be said that class `GarbageCan` is contravariant in its type parameter `A`. Next code snippet shows contravariant subtyping because `Item` is the super type of `PlasticItem`.

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
               A  <:                B
VendingMachine[B] <: VendingMachine[A]
{% endhighlight %}

If `A` is a subtype of `B` then `VendingMachine[B]` should be a subtype of `VendingMachine[A]`. This property is called contravariant subtyping.

### Use cases for contravariant type parameter ###

Contravariant type parameter is usually used as method argument type, so naturally contravariance is most commonly used in consumers (types that accept something). Just like with the covariance, Scala compiler prevents from the use of contravariant type parameter in positions that could lead to potential errors. If contravariant type parameter was used in illegal position such as method return type, Scala compiler would have reported an error. 

Typical use of contravariant type parameter is applied in the [example implementation of `GarbageCan`](https://github.com/kamkor/covariance-and-contravariance-examples/blob/master/src/main/scala/kamkor/contravariance/garbagecan/GarbageCan.scala). Code snippet below shows `GarbageCan` trait.

{% highlight scala %}
/** GarbageCan that is contravariant in its type parameter. */
trait GarbageCan[-A] {

  /** Puts item into this garbage can */
  def put(item: A): Unit

  /** Puts items into this garbage can */
  def putAll(items: List[A]): Unit

  /** Returns current number of items in the garbage can */
  val itemsCount: Int

}
{% endhighlight %}

## Real world use cases ##

Covariance and contravariance is actually quite common in functional style programming. This section shows example use cases from real APIs used by many developers on daily basis.

### Function type (T1) => R ###

One of the most common type used in the functional programming style is [function (T1) => R](http://www.scala-lang.org/api/current/#scala.Function1). 

{% highlight scala %}
// a bit simplified source code from Scala API
trait Function1[-T1, +R] extends AnyRef { 
  def apply(v1: T1): R
}
{% endhighlight %}

In this type both contravariance and covariance is used. `T1` is contravariant type parameter because it is used as `apply` input type and `R` is covariant type parameter because it is used as `apply` return type. This makes type `Function1` very generic. Let's examine how contravariance in `Function1` helps you achieve more with less code. Below code snippet presents music instruments model.

{% highlight scala %}
class MusicInstrument {
  val productionYear: Int
}
case class Guitar(val productionYear: Int) extends MusicInstrument   
case class Piano(val productionYear: Int) extends MusicInstrument
{% endhighlight %}

Next code snippet shows a function that returns true if MusicInstrument is vintage.

{% highlight scala %}
val isVintage: (MusicInstrument => Boolean) = _.productionYear < 1980
{% endhighlight %}

This function can be used to [filter](http://www.scala-lang.org/api/current/index.html#scala.collection.immutable.List@filter(p:A=>Boolean):Repr) both `List[Piano]` and `List[Guitar]`. If T1 from function (T1) => R, was not a contravariant type parameter, then two isVintage function would have to be implemented, one for 'Piano' and one for 'Guitar'. 

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

### Immutable container types - Option, Collections ###

Scala immutable container types such as [`Option`](http://www.scala-lang.org/api/current/#scala.Option), [`List`](http://www.scala-lang.org/api/current/#scala.collection.immutable.List) are covariant in its type parameter. If this wasn't the case, then in the code snippet below, separate `getPrices` method for each `MusicInstrument` would have to be implemented. However, with the covariance, one method is enough.

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

This library has been implemented in many languages, also in Scala in [RxScala](http://reactivex.io/rxscala/) project. Quote . 

This library makes heavy use of covariance and contravariance. 

 a library for composing asynchronous and event-based programs by using observable sequences.
Asynchronous

## Use-site and declaration-site covariance and contravariance ##

## Summary ##

Covariance and contravariance is something that many take for granted. It is especially relevant now because of the functional style programming that has gained a lot of popularity in the recent years. If covariance and contravariance would be suddenly turned off, a lot of code wouldn't compile any more.

It is important that both library developers and users understand concepts of covariance and contravariance. Covariance and contravariance makes libraries more generic and lets their aware users achieve more functionality with less code. 

From the perspective of library developer it is easiest to remember that covariant type parameter should be most often used as return type of a method and contravariant type parameter as a method argument type. Also there are use restrictions of covariant and contravariant type parameters which are checked by Scala compiler.

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

