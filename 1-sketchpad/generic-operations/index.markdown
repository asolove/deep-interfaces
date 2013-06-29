---
layout: default
title: "Generic operations"
---

<h2 class="pre-header">Part I: Sketchpad</h2>
# Generic operations for Sketchpad's view layer

Our first goal is to create a simple version of Sketchpad's view layer so that we can
  draw shapes and interact with them. This interface will help us understand and debug the
  underlying logic of constraint satisfaction and instance-creation.

### Data types

Our version of Sketchpad will start out with only a few data types: points, line segments,
  and circles. A point is defined by its x and y coordinates. Lines and circles are each defined
  by two points. We can define appropriate records for each type:

{%highlight clojure %}
(ns sketchpad.shapes)

(defrecord Point [x y])
(defrecord Line [p1 p2])
(defrecord Circle [center p2])
{% endhighlight %}

In the initial version of Sketchpad, the lines and circles had  
  pointers to the location in memory of their defining points. Our version will instead store a drawing
  as a mapping between object names and their data. Lines and circles can then store 
  the symbol names of the points they depend on. As we'll see later, these symbolic names will
  also be necessary when we add constraints and inheritance to the system. (The names themselves
  have no meaning and can be anything.)

Here is a simple drawing:

{% highlight clojure %}
(def drawing {
  :p1 (Point. 0 0)
  :p2 (Point. 10 10)
  :l1 (Line. :p1 :p2)
  :c1 (Circle. :p1 :p2) })
{% endhighlight %}

With this representation, many operations on individual lines or circles
  will also need access to the entire drawing's state. This will turn out to be helpful.</p>

### Generic operations

We would like to separate the mechanics of display and interaction from any
  knowledge about the objects themselves. We will define the needs of the interface
  as Clojure protocols, so that new types of geometric objects can be easily
  added later.

For now, the interface will need each shape to be able to draw itself to the canvas,
  to report how far it is from the cursor, and to move itself by a given amount as we 
  drag it around

The `Selectable` protocol defines the method `cursor-distance`,
  which returns a simple numeric value:

{% highlight clojure %}
(cursor-distance (Point. 0 0) [3 4]) ; => 5
{% endhighlight %}

The `Drawable` protocol defines the method `draw`, which returns nothing,
  but causes the shape to draw itself onto the canvas as a side-effect:

{% highlight clojure %}
(def point (Point. 1 1))
(def drawing {:p1 point})
(draw point context drawing) ; => null, but drawn to context
{% endhighlight %}

The `Moveable` protocol's `move!` method is the most peculiar. Moving a point is simple enough, and 
  could just return an updated version of the point. But when moving a line, we do
  not need to change anything about the line: it still starts and ends at points with the same names. Instead we need to move those two end-points. So `move!` returns a state patch, a map
  from names to updated versions of the changed objects.

{% highlight clojure %}
(def drawing {
  :p1 (Point. 1 1)
  :p2 (Point. 2 2)
  :l1 (Line. :p1 :p2) })

(move! (:p1 drawing) :p1 [5 5] drawing)
; => { :p1 (Point. 6 6)}

(move! (:l1 drawing) :l1 [5 5] drawing)
; => { :p1 (Point. 6 6) :p2 (Point. 7 7)}
{% endhighlight %}

#### Exercises

Before reading further, try out the following problems for yourself:

1. (Easy) Implement the `Moveable` protocol for each shape.
1. (Medium) Implement the `Selectable` protocol for each shape.
1. (Easy if you know canvas) Implement `Drawable` using the canvas API.

#### Answers

{% highlight clojure %}
(defprotocol Moveable
  "Objects that can be moved by direct interaction"

  (move! [item name dx dy universe] 
  "Returns a state patch for the universe with item moved"))

(extend-type Point 
  Moveable
  (move! [point name [dx dy] layer]
    {name { :x (+ x dx) :y (+ y dy) }}))

(extend-type Line
  Moveable
  (move! [_ name ds layer]
    (merge (move! (p1 layer) p1 ds layer)
           (move! (p2 layer) p2 ds layer))))

(extend-type Circle
  Moveable
  (move! [_ name ds layer]
    (merge (move! (center layer) center ds layer)
           (move! (p2 layer) p2 ds layer))))
{% endhighlight %}

[Answer for 2.]

[Answer for 3.]

### The interface