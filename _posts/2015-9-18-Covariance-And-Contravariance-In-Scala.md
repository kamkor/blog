---
layout: post
title: Covariance and contravariance in Scala
---
This post explains concepts of covariance and contravariance. It focuses on real world metaphors and on code examples that show how those concepts can be used in Scala. Metaphors are borrowed from Erik Meijer talk _Contravariance is the Dual of Covariance Implies Iterable is the Dual of Observable (Making Money Using Math)_ [Meijer2014].

## Covariance
<a name="covariance"></a>
First example of the covariance section uses model of drinks presented below:

![diagram]({{ site.baseurl }}/images/Covariance-And-Contravariance-In-Scala/drinks_model.png)

Imagine that your employer promised a new soft drink vending machine. Covariance means that in its place he can install a cola or tonic water vending machine, because both cola and tonic water are subtypes of soft drink. Let's turn this description into Scala code:

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

To sum up:

{% highlight scala %}
// Covariant subtyping
               A  <:                B
VendingMachine[A] <: VendingMachine[B]
{% endhighlight %}

If `A` is a subtype of `B` then `VendingMachine[A]` should be a subtype of `VendingMachine[B]`. This property is called covariant subtyping [Odersky2014].

### <a name="covariantuserestrictions"></a> Use restrictions of covariant type parameter

When a type parameter is declared as covariant, the number of places where it can be safely used is reduced (method argument type, field type etc.). Thankfully, Scala compiler prevents from the use of covariant type parameter in positions that could lead to potential errors. To understand why this is important, let's take a look at example from Java. Arrays in Java are covariant which means that this line compiles `Object[] arr = new String[3];` because `String` is a subtype of `Object`. This leads to potential runtime errors:

{% highlight java %}
String[] c1 = { "abc" };
Object[] c2 = c1;
// OOPS! Line below throws the runtime exception: ArrayStoreException.
// Reason is, that c2 is actually an instance of Array of Strings, and
// we try to update it with an Integer object.
c2[0] = 1;
{% endhighlight %}

[Scala fixes this by making its Arrays invariant](http://www.scala-lang.org/api/current/#scala.Array). Scala Array type parameter doesn't have covariance annotation (+ prefix), because if it had, the method `def update(i: Int, x: T): Unit` could lead to potential runtime errors. And if it actually had a + prefix, Scala compiler would have reported a compile error for `def update(i: Int, x: T): Unit` method declaration. So don't be afraid of using covariance because Scala compiler keeps you safe.

### <a name="legalcovariantpositions"></a> Legal positions of covariant type parameter ###

In general, covariant type parameter can be used for immutable field type, method return type and also for method argument type if method argument type has a lower bound. Because of those restrictions, covariance is most commonly used in producers (types that return something) and immutable types. Those rules are applied in the example [implementation of the Vending Machine](https://github.com/kamkor/covariance-and-contravariance-examples/blob/master/src/main/scala/kamkor/covariance/vendingmachine/VendingMachine.scala). Code snippet below shows `VendingMachine` trait.

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

`SoftDrink` is the common super type of both `TonicWater` and `Cola`, so `addAll` above returns a `VendingMachine` of `SoftDrink`.

#### Covariant type parameter as mutable field type

Covariant type parameter can be used for mutable field type only if the field has object private scope (`private[this]`). This is explained in Programming In Scala [Odersky2008]:

> Object private members can be accessed only from within the object in which they are defined. It turns out that accesses to variables from the same object in which they are defined do not cause problems with variance. The intuitive explanation is that, in order to construct a case where variance would lead to type errors, you need to have a reference to a containing object that has a statically weaker type than the type the object was defined with. For accesses to object private values, however, this is impossible.

Personally, I can't imagine there being many good use cases for using covariant type parameter for mutable field type. However, I have prepared an example that shows how rule above can be applied in Scala. Imagine that you are creating a sci-fi shooter game with different kinds of bullets:

{% highlight scala %}
trait Bullet
class NormalBullet extends Bullet
class ExplosiveBullet extends Bullet
{% endhighlight %}

Bullets are contained in the the ammo magazine as seen in the next code listing. Notice that class `AmmoMagazine` is covariant in its type parameter. It also has mutable field `bullets` which compiles because of object private scope. Every time `nextBullet` is invoked, bullets list is modified. Once ammo magazine is out of bullets, it can't refilled, and there's no way of introducing this feature into this class because [that would lead to potential runtime errors](#covariantuserestrictions).

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

This section is very similar to the previous one because contravariance is simply the opposite of covariance. Example of contravariance section uses model of items presented below:

![diagram]({{ site.baseurl }}/images/Covariance-And-Contravariance-In-Scala/items_model.png)

Imagine that you have to buy a garbage can for plastic items. Contravariance means that you can buy garbage can for item, because item is the super type of plastic item. In Scala, this can be expressed as follows:

{% highlight scala %}
class GarbageCan[-A] {
  // .. don't worry about implementation yet
}

def install(plasticItemsGC: GarbageCan[PlasticItem]): Unit = {
  // Installs garbage can for PlasticItem trash
}
{% endhighlight %}

`install` method can accept a `GarbageCan` of type `PlasticItem` or supertype of `PlasticItem` (`Item`). This is possible because type parameter A is prefixed with a -. It indicates that subtyping is contravariant in that parameter. Alternatively, it can be said that class `GarbageCan` is contravariant in its type parameter `A`. Next code snippet shows contravariant subtyping because `Item` is the supertype of `PlasticItem`.

{% highlight scala %}
// contravariant subtyping
install(new GarbageCan[Item])

// invariant
install(new GarbageCan[PlasticItem])
{% endhighlight %}

However, a `GarbageCan` of type `PlasticBottle` can't be passed to install method, because `PlasticBottle` is the subtype of `PlasticItem`. [That would be covariant subtyping](#covariance). 

{% highlight scala %}
// Compile error ! covariant subtyping
install(new GarbageCan[PlasticBottle])
{% endhighlight %}

To sum up:

{% highlight scala %}
// Contravariant subtyping
               A  <:                B
VendingMachine[B] <: VendingMachine[A]
{% endhighlight %}

If `A` is a subtype of `B` then `VendingMachine[B]` should be a subtype of `VendingMachine[A]`. This property is called contravariant subtyping.

Contravariant type parameter is most commonly used as method argument type. Just like with the covariance, Scala compiler prevents from the use of contravariant type parameter in positions that could lead to potential errors.

As an example, take a look at GarbageCan trait:

{% highlight scala %}
trait GarbageCan[-A] {

  /** Puts item into this garbage can */
  def put(item: A): Unit

}
{% endhighlight %}

If you used contravariant type parameter A in illegal postion such as method return type, Scala compiler would have reported an error. 

## Summary ##

I plan to create a follow up post where I explain the difference between declaration site variance (as seen in Scala) and use site variance (as seen in Java in the form of ? extends and ? super). I also would like to dive deeper into real world uses of covariance and contravariance - it really is very common nowadays, especially when writing functional style code. 

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

