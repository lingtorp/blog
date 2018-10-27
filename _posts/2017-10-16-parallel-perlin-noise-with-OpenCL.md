---
layout: post
title: "Parallel Perlin noise with OpenCL"
last_modified_at: 2017-10-16
tags:
  - opencl
  - perlin
---
<html>
<head>
    <meta charset="UTF-8">
    <link rel="stylesheet" media="all" href="normalize.css">
    <link rel="stylesheet" media="all" href="core.css">
    <link rel="stylesheet" media="all" href="style.css">
    <script type="text/javascript" src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML"></script>
</head>
<body data-document>&nbsp;</body>
</html>

Noise is awesome. GPUs are awesome. Let's but the first onto the latter. Great, let's go!

# Perlin noise
People are in general very confused about the Perlin noise algorithm. It is defined by Kenneth Perlin in his paper
from 1985, found [here](). He later in 2002 revised it and improved it significantly which is why the new algorith 
has a  new name; namely Simplex noise. The paper on the Simplex noise algorithm can be found [here](). 

# Gradients
Both Perlin noise and Simplex noise are algorithms which depend on a coordinate grid. The grid is used as a sort of
point of reference for the input coordinates.   

## Runtime complexity
Since we are generating an image every pixel is independent of one another. Therefore we can run the Perlin noise 
algorithm over all of the pixels in parallel which is super awesome. Lets call this _feature_ pixel-parallel.

It turns out that there are parts in the algorithm itself that are inherently independent of each other. We might
want to investigate whether or not this will yield any performance increases. We start by breaking about the 
algorithm into parts and analyze their dependencies on each other. We will represent this with a direct acyclic graph (DAG).

<img style="float: right;" src="
https://g.gravizo.com/svg?
 digraph G {
   main -> parse -> execute;
   main -> init;
   main -> cleanup;
   execute -> make_string;
   execute -> printf
   init -> make_string;
   main -> printf;
   execute -> compare;
 }
"/>

## Memory complexity
\\[ x = {-b \pm \sqrt{b^2-4ac} \over 2a} \\]

# OpenCL