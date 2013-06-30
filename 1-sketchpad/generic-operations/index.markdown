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
(defrecord Circle [center point])
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
  will also need access to the entire drawing's state. This will turn out to be helpful.

### Generic operations

We would like to separate the mechanics of display and interaction from any
  knowledge about the objects themselves. We will define the needs of the interface
  as Clojure protocols, so that new types of geometric objects can be easily
  added later.

For now, the interface will need each shape to be able to draw itself to the canvas,
  to report how far it is from the cursor, and to move itself by a given amount as we 
  drag it around.

#### Drawing

The first thing our interface needs to do is draw static shapes. The `draw` method
accepts a shape, a canvas drawing context, and the complete drawing state. It returns nothing
but as a side-effect it draws itself to the canvas.

We'll create a new namespace to keep all of our drawing code separate from other logic. 
Drawing these shapes to a canvas is quite easy:

{% highlight clojure %}
(ns sketchpad.drawable
  (:use [sketchpad.shapes :only [Point Line Circle]]))

(defprotocol Drawable
  "Objects that can be drawn to a browser canvas"
  (draw [item ctx drawing] 
  "Draw the object on the context for the provided drawing"))

(extend-type Point
  Drawable
  (draw [point ctx drawing]
    (drawCircle ctx { :stroke "#888" :fill "#999"
                      :x x :y y :r 2 })))
{% endhighlight %}

To draw a line or a circle, we'll first need to look up the coordinates of the two 
defining points from the drawing state:

{% highlight clojure %}
(extend-type Line
  Drawable
  (draw [line ctx drawing]
    (let [{x1 :x y1 :y} (drawing p1)
          {x2 :x y2 :y} (drawing p2)]
        (drawLine ctx { :x1 x1 :x2 x2 :y1 y1 :y2 y2 :w 2 }))))

(extend-type Circle
  Drawable
  (draw [line ctx universe]
    (let [{cx :x cy :y} (drawing center)
          {sx :x su :y} (drawing point)
          r (distance [cx cy] [sx sy])]
        (drawCircle ctx {:fill "transparent" :stroke "#999"
                         :strokeWidth 1 :x cx :y cy :r r }))))
{% endhighlight %}

The implementations of `draw` are terse because of two helper
methods that hide the stateful and verbose detail code for drawing to the canvas:

{% highlight clojure %}
(defn drawLine [context {:keys [x1 y1 x2 y2 w]}]
  (.beginPath context)
  (set! (.-lineWidth context) (or w 1))
  (set! (.-strokeStyle context) "#999")
  (.moveTo context x1 y1)
  (.lineTo context x2 y2)
  (.stroke context))

(defn drawCircle [context {:keys [x y r fill stroke strokeWidth]}]
  (.beginPath context)
  (.arc context x y r 0 (* 2 Math/PI) false)
  (set! (.-fillStyle context) (or fill "transparent"))
  (.fill context)
  (set! (.-lineWidth context) (or strokeWidth 1))
  (set! (.-strokeStyle context) stroke)
  (.stroke context))
{% endhighlight %}

We can now begin to write the interface code and display our first shapes:

{% highlight clojure %}
(ns sketchpad.ui
  (:use [sketchpad.shapes :only [Point Line Circle]]
        [sketchpad.drawable :only [Drawable draw]]))

(defn drawables [drawing]
  (filter (fn [[name item]]] (satisfies? Drawable item)) drawing))

(defn draw-all [drawing ctx]
  (.clearRect ctx 0 0 1000 1000)
  (doseq [[name item] (drawables universe)]
    (draw item ctx drawing))

(defn ^:export main []
  (let [canvas (js/document.getElementById "canvas")
        ctx (.getContext canvas "2d")
        drawing { :p1 (Point. 10 10) 
                  :p2 (Point. 10 20) 
                  :l1 (Line. :p1 :p2)
                  :c1 (Circle. :p1 :p2) }]
    (draw-all drawing)))
{% endhighlight %}

#### Selecting

Our interface will need to track the user's cursor and decide which shape it is closest to.
The `cursor-distance` method gives the distance between the shape and a provided set of coordinates:

{% highlight clojure %}
(cursor-distance (Point. 0 0) [3 4]) ; => 5
{% endhighlight %}

Implementing `cursor-distance` for points is simple enough:

{% highlight clojure %}
(defn distance [a b]
  (->> (map - a b) (map #(Math.pow % 2)) (reduce +) Math.sqrt))

(extend-type Point
  Selectable
  (cursor-distance [point [cx cy] drawing]
    (distance [x y] [cx cy])))
{% endhighlight %}

For a circle, the math is simple enough, but we will need to first look up the coordinates
of the circle's two defining points in the provided drawing data:

{% highlight clojure %}
(extend-type Circle
  Selectable
  (cursor-distance [circle [x y] drawing]
    (let [{cx :x cy :y} (drawing center)
          {px :x py :y} (drawing point)
          r (distance [cx cy] [sx sy])
          d (distance [cx cy] [x y])]
      (Math.abs (- d r)))))
{% endhighlight %}

Finding the distance between a point and a line segment is a slightly more complicated matter.


#### Moving

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

#### Answers

{% highlight clojure %}
(defprotocol Moveable
  "Objects that can be moved by direct interaction"

  (move! [item name dx dy universe] 
  "Returns a state patch for the universe with item moved"))

(extend-type Point 
  Moveable
  (move! [point name [dx dy] drawing]
    {name { :x (+ x dx) :y (+ y dy) }}))

(extend-type Line
  Moveable
  (move! [_ name ds drawing]
    (merge (move! (p1 drawing) p1 ds drawing)
           (move! (p2 drawing) p2 ds drawing))))

(extend-type Circle
  Moveable
  (move! [_ name ds drawing]
    (merge (move! (center drawing) center ds drawing)
           (move! (p2 drawing) p2 ds drawing))))
{% endhighlight %}

