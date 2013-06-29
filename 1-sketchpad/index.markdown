---
layout: default
title: "Sketchpad"
---

<h2 class="pre-header">Part I</h2>
<h1>Sketchpad: the first graphical user interface</h1>

<blockquote class="teaser">
	Sketchpad encountered a critical challenge that remains central to human-
	computer interaction. Sutherland's original aim was to make computers accessible to new classes of user (artists and draughtsmen among others), while
	retaining the powers of abstraction that are critical to programmers. ...
	Sutherland's attempt to
	remove the division between users and programmers was not the only system that, 
	in failing to do so, provided the imaginative leap to a new programming paradigm.
	<cite>Alan Blackwell and Kerry Rodden, in the preface to the the <a href="http://www.cl.cam.ac.uk/techreports/UCAM-CL-TR-574.pdf">2003 republication</a> of Sutherland's thesis.</cite>
</blockquote>
	

<p>Ivan Sutherland's Sketchpad was the first graphical user interface.
	With Sketchpad, a user could directly manipulate points, lines and circles
	to lay out a geometric construction, a circuit, or a two-dimensional bridge truss.
	Once the picture was close to the correct shape, the user could begin to apply
	mathematical constraints, commanding two lines to be parallel, or a text label 
	to take on the value of the radius of a circle. The Sketchpad system would then
	maintain these constraints dynamically, even as the user made manual adjustments.</p>
	
<p>Watch Alan Kay describe Sketchpad and narrate a video of it in use:</p>	
	
<iframe width="550" height="400" src="http://www.youtube.com/embed/mOZqRJzE8xg" frameborder="0" allowfullscreen="allowfullscreen" style="margin-bottom:30px;">&nbsp;</iframe>
	
<h2>Lessons from Sketchpad</h2>
	
<p>Besides its historical interest as the first GUI, Sketchpad
	also demonstrates several important techniques for building powerful interfaces:</p>
	
<ol>
	<li><strong>Generic operations</strong>: though object-oriented programming did not yet
		exist, Sutherland created a similar system in which the same operation could be applied
		to different types of data, without respect to the specific type of the piece of data.
		Many algorithms were greatly simplified by being able to call a generic operation and
		have the concrete steps performed dispatch on the type of the data.</li>
	<li><strong>Constraint satisfaction</strong>: when users can describe the properties they want
		to hold, rather than construct the solution themselves, the computer can use its computational
		power to allow the user to focus on the larger picture.</li>
	<li><strong>Prototypical inheritance</strong>: after creating a drawing, users could import
		it whole into another drawing. The copied drawing could include complex constraints and 
		join-points to allow it to connect to other drawings. Changes in the first drawing would then 
		immediately propagate to all of its uses elsewhere.</li>
</ol>

<h2>Recreating Sketchpad</h2>

<p>Our version of Sketchpad will support only a few shapes and constraint types, but will
demonstrate the power and generality of the original. We will be drawing in the browser 
using the canvas API and simple DOM events.</p>

<p><strong>Next: <a href="/1-sketchpad/generic-operations/">Generic operations for Sketchpad's view layer</a></strong></p>
