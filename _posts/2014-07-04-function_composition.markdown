---
layout: post
title:  "Function composition"
date:   2014-07-04 20:12:26
categories: functional programming
---

Function composition is a great tool when you want to combine many functions to output a new function. This new function will take a parameter to output the result. Function composition takes the form (b -> c) -> (a -> b) -> a -> c. In other words this takes 2 functions; one function with (a -> b) and another with (b -> c). A quick observation will reveal that the output of second function which is b is the parameter to the first function. If this matches then we can compose two functions.

Here's a contrived example in Haskell:
The code below checks whether an element is present in a list of numbers. To make the example interesting, we will lift the numbers in a list to Maybe and then pass this list containing [Maybe a] to findNElem. 

The lift' function will take a number and transforms to Maybe.

{% highlight haskell %}
lift' :: (Num a) => [a] -> [Maybe a]
lift' = map (\f -> Just f)
{% endhighlight %}



The fromJust function is a helper function to transform from Maybe a to a. The code is pretty straight forward.

{% highlight haskell %}
fromJust :: (Num a) => Maybe a -> a
fromJust (Just x) = x
{% endhighlight %}

This function will take a number and checks whether that number is present in the list.

{% highlight haskell %}
findNElem :: (Num a , Eq a) => a -> [Maybe a] -> Bool
findNElem a [] = False
findNElem a (x:xs) = if a == fromJust x then
                        True
                     else 
                        findNElem a xs
{% endhighlight %}

If we take a look without using function composition it looks something like this:

findNElem 3 (lift' [1,2,3])

But this has a limitation of extra parantheses and passing of list [1,2,3] to lift'. Instead if the list is not yet known and all we wanted to check if 3 is present in a list which will be known in future then we can do something like below.

{% highlight haskell %}
 λ> let fcompl = findNElem 3 . lift'
 λ> :t fcompl
fcompl :: [Integer] -> Bool
{% endhighlight %}

Going from the above definition: (b -> c) -> (a -> b) -> (a -> c)

The type of lift':

{% highlight haskell %}
 λ> :t lift'
lift' :: Num a => [a] -> [Maybe a]
{% endhighlight %}

Here:
(a  ->  b) 
is equivalent to
[a] -> [Maybe a]

The type of findNElem:

{% highlight haskell %}
λ> :t findNElem 3
findNElem 3 :: (Eq a, Num a) => [Maybe a] -> Bool
{% endhighlight %}

Here:
(b -> c)
is equivalent to
[Maybe a] -> Bool

So the final type should look like:
(a -> c)
[a] -> Bool
which in our case is
[Integer] -> Bool

So to finally arrive at this composed type we use (.) to compose functions.

Here's the type of (.):

{% highlight haskell %}
λ> : (.)
(.) :: (b -> c) -> (a -> b) -> a -> c
{% endhighlight %}

which is same as what was defined aboove.

{% highlight haskell %}
λ> let fcompl = findNElem 3 . lift'
λ> :t fcompl
fcompl :: [Integer] -> Bool
{% endhighlight %}

Finally, we run it check if the element exists:

{% highlight haskell %}
λ> fcompl [1,2,3]
True
{% endhighlight %}

Here's the same example but in Scala:

{% highlight scala %}
def lift(ls: List[Int]) = ls map { x => Option(x)}
{% endhighlight %}

{% highlight scala %}
scala> :type lift _
List[Int] => List[Option[Int]]
{% endhighlight %}

{% highlight scala %}
def findNElem(elem: Int, ls: List[Option[Int]]): Boolean = ls match {
    case Nil => false
	case x :: xs => if (elem == x.get) true else findNElem (elem, xs)
}
{% endhighlight %}

The point worth mentioning here is that we curry the function in Scala because Scala doesn't provide currying by default. 

{% highlight scala %}
val findElemCurried = (funcompose.findNElem _).curried
{% endhighlight %}

If we look at the type we see it returns a Function1 which takes a function which takes a Int parameter and a function.

{% highlight scala %}
scala> val findElemCurried = (funcompose.findNElem _).curried
findElemCurried: Int => (List[Option[Int]] => Boolean) = <function1>
{% endhighlight %}

{% highlight scala %}
val fcompl = findElemCurried(3) compose funcompose.lift _

scala> :type findElemCurried(3)
List[Option[Int]] => Boolean
{% endhighlight %}

{% highlight scala %}
fcompl(List[1,2,3])
{% endhighlight %}

<h4>Conclusion</h4>

Function composition can become overwhelming if we compose too many functions. But, for some cases which needs reusability, this is a great tool. Haskell supports currying by default and hence the code looks pretty and succinct compared to Scala counterpart.