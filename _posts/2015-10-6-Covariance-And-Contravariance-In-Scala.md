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

The method `install` accepts a `VendingMachine` of type `SoftDrink` or subtypes of `SoftDrink` (`Cola` and `TonicWater`). This is possible because type parameter `A` is prefixed with a +. It indicates that subtyping is covariant in that parameter. Alternatively, it can be said that class `VendingMachine` is covariant in its type parameter `A`. Next code snippet shows covariant subtyping because `Cola` and `TonicWater` are subtypes of `SoftDrink`.

{% highlight scala %}
// covariant subtyping
install(new VendingMachine[Cola])
install(new VendingMachine[TonicWater])
// invariant
install(new VendingMachine[SoftDrink])
{% endhighlight %}

However, a `VendingMachine` of type `Drink` can't be passed to the method `install` because `Drink` is a supertype of `SoftDrink`. That would be contravariant subtyping. [Contravariance is explained later](#contravariance).

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

When a type parameter is declared as covariant, the number of positions where it can be safely used is reduced. Thankfully, Scala compiler prevents from the use of covariant type parameter in positions that could lead to potential errors. Next example, this time from Java, shows why this is important. 

Unfortunately, arrays in Java are covariant. This allows code below to compile because `String` is a subtype of `Object`. Therefore, covariant subtyping can be applied.

{% highlight java %}
Object[] objects = new String[] { "abc" };
{% endhighlight %}

If `objects` would be modified with value of type other than `String` then `ArrayStoreException` would be thrown.

{% highlight java %}
String[] strings = { "abc" };
Object[] objects = strings;
// OOPS! Line below throws runtime exception: ArrayStoreException.
// Reason is that objects is actually an instance of 
// Array of Strings and we try to update it with an Integer.
objects[0] = 1;
{% endhighlight %}

Scala fixes this problem by making its [Array](http://www.scala-lang.org/api/current/#scala.Array) type invariant in its type parameter.

{% highlight scala %}
// Simplified Array from Scala
final class Array[T](_length: Int) {
  // arr.update(i, x) is equivalent to arr(i) = x
  def update(i: Int, x: T): Unit = ...
  // ...
}
{% endhighlight %}

Scala's `Array` type parameter doesn't have covariance annotation (+ prefix). If it had then the method `def update(i: Int, x: T): Unit` could lead to potential runtime errors and in Scala it wouldn't even compile.

### <a name="legalcovariantpositions"></a> Legal positions of covariant type parameter

In general, covariant type parameter can be used as immutable field type, method return type and also as method argument type if the method argument type has a lower bound. Because of those restrictions, covariance is most commonly used in producers (types that return something) and immutable types. Those rules are applied in the following implementation of vending machine.

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

The method `def addAll[B >: A](newItems: List[B]): VendingMachine[B]` has very useful characteristic, that is, a lower bound makes the method `addAll` very flexible as seen below.

{% highlight scala %}
val colasVM: VendingMachine[Cola] = 
                    new VendingMachine(List(new Cola, new Cola))
val softDrinksVM: VendingMachine[SoftDrink] = 
                    colasVM.addAll(List(new TonicWater))
{% endhighlight %}

`SoftDrink` is the common supertype of both `TonicWater` and `Cola`, so the method `addAll` above returns instance of `VendingMachine` of type `SoftDrink`. 

#### <a name="variancemutablefieldtype"></a> Covariant (and contravariant) type parameter as mutable field type

Type parameter with variance annotation (covariant + or contravariant -) can be used as mutable field type only if the field has object private scope (`private[this]`). This is explained in Programming In Scala [Odersky2008].

> Object private members can be accessed only from within the object in which they are defined. It turns out that accesses to variables from the same object in which they are defined do not cause problems with variance. The intuitive explanation is that, in order to construct a case where variance would lead to type errors, you need to have a reference to a containing object that has a statically weaker type than the type the object was defined with. For accesses to object private values, however, this is impossible.

Next example shows how rule above could be applied in practice. Imagine that you are creating a sci-fi shooter game where player can shoot with different kinds of bullets.

{% highlight scala %}
trait Bullet
class NormalBullet extends Bullet
class ExplosiveBullet extends Bullet
{% endhighlight %}

Bullets are contained in the the ammo magazine as seen in the next code listing. Notice that class `AmmoMagazine` is covariant in its type parameter `A`. It also contains mutable field `bullets` which compiles because of object private scope. Everytime the method `giveNextBullet` is invoked, bullet from the `bullets` list is removed. `AmmoMagazine` can't be refilled with bullets and there is no way of introducing this feature into this class because [that would have led to potential runtime errors](#covariantuserestrictions).

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

Class `AmmoMagazine` is used in the `Gun` class. `Gun` has method `reload` which thanks to the covariant subtyping can be reloaded with both `AmmoMagazine[NormalBullet]` and `AmmoMagazine[ExplosiveBullet]`, and in general with `AmmoMagazine` of any subtype of `Bullet`.

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

The method `setGarbageCanForPlastic` accepts a `GarbageCan` of type `PlasticItem` or supertype of `PlasticItem` (`Item`). This is possible because type parameter `A` is prefixed with a -. It indicates that subtyping is contravariant in that parameter. Alternatively, it can be said that class `GarbageCan` is contravariant in its type parameter `A`. Next code snippet shows contravariant subtyping because `Item` is the supertype of `PlasticItem`.

{% highlight scala %}
// contravariant subtyping
setGarbageCanForPlastic(new GarbageCan[Item])

// invariant
setGarbageCanForPlastic(new GarbageCan[PlasticItem])
{% endhighlight %}

However, a `GarbageCan` of type `PlasticBottle` can't be passed to the method `setGarbageCanForPlastic`, because `PlasticBottle` is the subtype of `PlasticItem`. [That would be covariant subtyping](#covariance). 

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

### Use cases for contravariant type parameter

Contravariant type parameter is usually used as method argument type, therefore, contravariance is most commonly associated with consumers (types that accept something). It can also be used as mutable field type if the field has object private scope as explained [before](#variancemutablefieldtype). Scala compiler prevents from the use of contravariant type parameter in positions that could lead to potential errors. If contravariant type parameter would have been used in illegal position such as method return type then Scala compiler would have reported an error. Example use of contravariant type parameter is applied in the following implementation of garbage can.

{% highlight scala %}
class GarbageCan[-A] {

  // compiles because of object private scope
  private[this] var items: List[A] = List.empty

  def put(item: A): Unit = this.items :+= item

  def putAll(items: List[A]): Unit = this.items ++= items

  def itemsCount: Int = this.items.size

}
{% endhighlight %}

## Real world examples of covariance and contravariance

Covariance and contravariance is very common in functional style programming. Many libraries used by developers on a daily basis use variance annotations to make library types more flexible by allowing the use of covariant and contravariant subtyping.

### Function type (T) => R ###

One of the most common type used in the functional programming style is [function (T) => R](http://www.scala-lang.org/api/current/#scala.Function1). 

{% highlight scala %}
// a bit simplified source code from Scala API
trait Function1[-T, +R] extends AnyRef { 
  def apply(v1: T): R
}
{% endhighlight %}

In this type both contravariance and covariance is used. `T` is contravariant type parameter because it is used as `apply` argument type and `R` is covariant type parameter because it is used as `apply` return type. This makes type `Function1` very flexible. Let's examine how contravariance in `Function1` helps its users achieve more with less code. Below code snippet presents music instruments model.

{% highlight scala %}
trait MusicInstrument {
  val productionYear: Int
}
case class Guitar(productionYear: Int) extends MusicInstrument   
case class Piano(productionYear: Int) extends MusicInstrument
{% endhighlight %}

Next code snippet shows a function that returns true if `MusicInstrument` is vintage.

{% highlight scala %}
val isVintage: (MusicInstrument => Boolean) = _.productionYear < 1980
{% endhighlight %}

Method `isVintage` can be used to [filter](http://www.scala-lang.org/api/current/index.html#scala.collection.immutable.List@filter(p:A=>Boolean):Repr) both `List[Piano]` and `List[Guitar]`. This is possible because contravariant subtyping can be applied. If `T` from function `(T) => R` was not a contravariant type parameter then two versions of `isVintage` function would have to be implemented, one for `Piano` and one for `Guitar`. See it in action in the code listing below.

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

Example above only took advantage of contravariant subtyping but covariant subtyping can be just as useful.

### Immutable container types - Option, Collections ###

Scala immutable container types such as [`Option`](http://www.scala-lang.org/api/current/#scala.Option) and [`List`](http://www.scala-lang.org/api/current/#scala.collection.immutable.List) are covariant in their type parameter. If this wasn't the case then in the code snippet below, separate `getPrices` method for each subtype of `MusicInstrument` would have to be implemented. However, with the use of covariant subtyping, one method is enough.

{% highlight scala %}
def getPrices(instruments: List[MusicInstrument]) = {
  // returns prices of instruments    
}

val guitars: List[Guitar] = List(new Guitar(1966), new Guitar(1988))  
// covariant subtyping because Guitar is a subtype of MusicInstrument
val guitarsPrices = getPrices(guitars)
{% endhighlight %}

### RxScala ###

Asynchronous and event-based programs have gained a lot of popularity in the recent years. It used to be really challenging to write such programs but with the arrival of the [Reactive Extensions](http://reactivex.io/) things became much easier. Quote below explains what are Reactive Extensions in short [Reactivex.io].

> ReactiveX is a library for composing asynchronous and event-based programs by using observable sequences.
>
>It extends the observer pattern to support sequences of data and/or events and adds operators that allow you to compose sequences together declaratively while abstracting away concerns about things like low-level threading, synchronization, thread-safety, concurrent data structures, and non-blocking I/O.

Reactive extensions have been implemented in many languages, also in Scala in project [RxScala](http://reactivex.io/rxscala/). What does it have to do with this post? This library makes heavy use of covariance and contravariance. Take look at [`Observable`](http://reactivex.io/rxscala/scaladoc/index.html#rx.lang.scala.Observable) and [`Observer`](http://reactivex.io/rxscala/scaladoc/index.html#rx.lang.scala.Observer) which I think are two most important types of this library. Those types are examined next and a new advanced concept called flipped classification is introduced.

#### Flipped classification

{% highlight scala %}
// very simiplified version of Observable
trait Observable[+T] {  
  def subscribe(observer: Observer[T]): Subscription  
  def map[R](func: T => R): Observable[R]  
  // much more  
}
{% endhighlight %}

`Observable` is a type that produces values of type `T` and is covariant in that type parameter. However, if you look at method `subscribe` you might think that it uses type `T` in illegal position. After all, it was explained before that if type parameter is declared as covariant then it can be used as method argument type if it has a lower bound (`[B >: T]`). So the question is, why `subscribe` above compiles? The answer is flipped classification. Method `subscribe` accepts parametrized type `Observer` as an argument. The trick is that `Observer` is contravariant in its type parameter `T`.

{% highlight scala %}
// simplified version of Observer
trait Observer[-T] {
  def onNext(value: T): Unit  
  // .. more methods
}
{% endhighlight %}

Flipped classification also applies to `map` method of `Observable`. Method `map` accepts function `(T => R)` as an argument, that is `Function1[-T, +R]`. `Function1` is contravariant in type parameter `T` so flipped classification is also applied by the Scala compiler. 

Flipped classification is explained in more detail in _The fast track_ section of Type Parameterization chapter in [Programming In Scala](http://www.artima.com/pins1ed/type-parameterization.html). 

## Use-site and declaration-site variance

Only declaration-site variance has been described so far. It is defined during the declaration of the type with a + prefix for covariant type parameter and a - prefix for contravariant type parameter.

Another kind of variance is use-site variance which is defined by the user of the type. It is best explained with an example. `VendingMachine` below is invariant in its type parameter `A`. 

{% highlight scala %}
class VendingMachine[A]
{% endhighlight %}

If the user of the type `VendingMachine` would like to use covariant subtyping then he would have to define covariance himself using upper bound.

{% highlight scala %}
def install(softDrinkVM: VendingMachine[_ <: SoftDrink]): Unit = {
  // Installs soft drink vending machine
}
// covariant subtyping
install(new VendingMachine[Cola])
{% endhighlight %}

If the method `install` was defined as `install(softDrinkVM: VendingMachine[SoftDrink]): Unit` then code above wouldn't compile because it would have required the types to match exactly. That is, the method `install` would have accepted only values of type `VendingMachine[SoftDrink]`. 

### Use-site variance readability and usage issues

Personally I think that in contrast to declaration-site variance, use-site variance makes code harder to read. Worst of all, use-site variance has to be often exposed in public API. Consider Java which only supports use-site variance for generics. Java 8 introduced lambdas and added a lot of new functional style types to its standard library. Those types often use covariance and contravariance to give their users more flexibility. The price of this flexibility is API that can be unpleasant to read. Listing below shows few example methods of new types added in Java 8. Return types are ommited.

{% highlight java %}
// from CompletableFuture
thenCombine(
  CompletionStage<? extends U> other,
  BiFunction<? super T,? super U,? extends V> fn)
    
// from Collectors
groupingBy(
  Function<? super T,? extends K> classifier, 
  Collector<? super T,A,D> downstream)
toMap(
  Function<? super T,? extends K> keyMapper, 
  Function<? super T,? extends U> valueMapper)
  
// from Function
compose(Function<? super V,? extends T> before)
{% endhighlight %}

In functional style programming it is very usual to have higher order functions, that is, functions that take function as an argument. In Java 8, everytime a higher order function is defined, the function taken as an argument should have use-site variance annotations: `Function<? super T,? extends K>`. This is really cumbersome but as Java programmers we have to live with it. Things would be much easier and more pleasant if Java featured declaration-site variance.

### Use-site variance may reduce functionality of the type

I believe that there is even a worse problem with use-site variance. Take a look at class `Box` defined in the next code listing. `Box` uses type parameter `A` as both output and input, so it can't be declared by the creator of the class as neither covariant nor contravariant. 

{% highlight scala %}
class Box[A]() {

  private var thing: A = _
  
  def retrieve: A = thing
  
  def put(thing: A) = this.thing = thing
  
}
{% endhighlight %}

Code below creates instances of `Box` of type `SoftDrink`. It puts `Cola` into it and then retrieves it.

{% highlight scala %}
val softDrinkBox: Box[SoftDrink] = new Box()
softDrinkBox.put(new Cola)
val softDrink: SoftDrink = box.retrieve
{% endhighlight %}

If the user would have liked to use covariant subtyping with class `Box` then he would have to add variance annotations himself. However, this would have made method `put` no longer usable because it wouldn't be possible for the compiler to figure out the correct type of the argument of the method `put`.

{% highlight scala %}
val softDrinkBox: Box[_ <: SoftDrink] = new Box()
// type mismatch; found : Cola required: _$4
// softDrinkBox.put(new Cola)
val softDrink: SoftDrink = softDrinkBox.retrieve
{% endhighlight %}

On the other hand, if the user would have liked to use contravariant subtyping using use-site variance then he would have made the method `retrieve` mostly unusable. That is, any type safety would be gone. User could only retrieve something like `Any`.

{% highlight scala %}
val softDrinkBox: Box[_ >: SoftDrink] = new Box()
softDrinkBox.put(new Cola)
// can't retrieve and assign to type SoftDrink 
val softDrinkMaybe: Any = softDrinkBox.retrieve
{% endhighlight %}

Java programmers sometimes have to make a collection like `java.util.List` covariant or contravariant in its type parameter. The same issues as with `Box` class apply in this case.

{% highlight java %}
// you can't put elements into numbersCov but only retrieve them
final java.util.List<? extends Number> numbersCov = getCovList();
// only null can be put in
numbersCov.add(null);
final Number n = numbersCov.get(0);

// you can put elements into numbersContr but can't retrieve them
final java.util.List<? super Number> numbersContr = getContrList();
numbersContr.add(5);
// can be only assigned to Object
final Object o = numbersContr.get(0);
{% endhighlight %}

The fact that use-site variance can make some functionality of the type completely unusable can suprise a lot of users (programmers). With declaration-site variance, Scala compiler makes sure that variance annotations are only used when it makes sense, that is, only if they add more flexibility for the user. Finally, declaration-site variance makes APIs and code much cleaner than use-site variance.

## Summary ##

It is important that both developers of types and their users understand covariance and contravariance. Correct use of variance makes types more flexible and lets their users achieve more functionality with less code. 

From the perspective of type developer it is easiest to remember that covariant type parameter should be most often used as output type (in producers) and contravariant type parameter as input type (in consumers). If type parameter has to be used as both output and input type then it should be invariant. However, remember that there are exceptions, for example, covariant type parameter can be used as method argument type if lower bound is used.

From the perspective of type user it is best to remember rules below. First, covariant subtyping.

{% highlight scala %}
// Covariant subtyping (Vending machine metaphore)
               A  <:                B
VendingMachine[A] <: VendingMachine[B]
{% endhighlight %}

And next, contravariant subtyping which is the opposite of covariant subtyping.

{% highlight scala %}
// Contravariant subtyping (Garbage can metaphore)
           A  <:            B
GarbageCan[B] <: GarbageCan[A]
{% endhighlight %}

If you forget rules above, try to remind yourself of vending machine and garbage can metaphors. Last of all, all code examples from this post and more are available at [github](https://github.com/kamkor/covariance-and-contravariance-examples).

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
Parametrized type                | `List[String]`               | `List<String>`
Actual type parameter            | `String`                     | `String`
Generic type                     | `List<A>`                    | `List<E>`
Formal type parameter            | `A`                          | `E```
Unbounded wildcard type          | `List[_]`                    | `List<?>`
Raw type             | `List`             | `List`
Type parameter with lower bound  | `[A >: Number]`              | `<E super Number>`
Type parameter with upper bound  | `[A <: Number]`                    | `<E extends Number>`
Wildcard type with lower bound   | `[_ >: Number]`          | `<? super Number>`
Wildcard type with upper bound   | `[_ <: Number]`          | `<? extends Number>`
Recursive type bound       | `[A <: Ordered[A]]`        | `<T extends Comparable<T>>`
Type constructor         | `List, constructs List[Int] etc`   | `Same as in Scala`
Variance annotation              | `+ or - i.e. [+A] or [-A]`          | `not supported`
Covariance annotation            | `+ i.e. [+A]`                       | `not supported`
Contravariance annotation        | `- i.e. [-A]`                       | `not supported`
