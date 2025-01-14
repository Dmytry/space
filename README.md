# Relativistic spaceflight simulator

### [View it here](https://dmytry.github.io/space)

![A screenshot showing relativistic effects](https://github.com/Dmytry/space/blob/master/article_img_1.png?raw=true)

Special thanks to Mr.doob for three.js (WebGL wrapper and scene graph renderer) and Silvio Paganini for help with adapting my bloom effect to three.js

This is a relativistic spaceflight simulator which shows what it would look like if you were on a 
spaceship traveling close to the speed of light. 
It is loosely based on my original code from 2012 implementing relativistic effects in my game The Polynomial. 

It simulates relativistic boost (for spaceship motion), relativistic aberration and Doppler effect (both frequency and intensity). It also simulates cosmic microwave background radiation.

The view is displayed as 1 year = 1 second timelapse. 

The code is currently a bit of a mess and needs a cleanup.

The general development philosophy is to make something that works now and will still work in 10 years. Consequently, the javascript is written directly rather than using a translation layer (since a translation layer could become unsupported and old versions may become difficult to run), node.js is not used, etc.

Copyright: 2020 Dmytry Lavrov. Website: http://dmytry.com

License: Attribution-ShareAlike CC BY-SA https://creativecommons.org/licenses/by-sa/4.0/legalcode for the index.html

# Explanations

I am going to go into a bit of detail of how I implemented relativistic rendering, complete with the Doppler Effect. I'm not going to discuss the source code verbatim, but make a few simplifications in the snippets.

I'm also going to make an assumption that you have some basic knowledge of special relativity, as it is a complex topic which is difficult to explain alongside all the rendering complications; it is much easier to first learn it in simple cases where velocities are pointed along the x axis, than in a renderer that handles arbitrary rotations.

First, lets establish some conventions. For rendering stars, it is most convenient to measure distances in light-years and time in years. Light travels one light-year per year, so the speed of light is then simply 1, which helps write equations concisely and without cluttering everything with c and c squared.
Transformations

3D transformations in computer graphics are conventionally done using 4x4 matrices and homogeneous coordinates (positions of form x,y,z,1).

Special relativity, however, works with spacetime coordinates. The typical convention in physics is to put the time first, but when you're working with GLSL it is more convenient to put the time last, so that XYZ refer to position, and W to time.

Instead of transforming a 4-vector x,y,z,1 using a model-view matrix , you would transform a spacetime 4-vector x,y,z,t . And instead of using a regular model-view matrix, you would use a Lorentz matrix (which can combine relativistic transformations and ordinary rotations).

In our case, we want to transform light as it hits the camera on the spaceship. For a star, a four-coordinate would consist of star's relative position, and the time at which the light was emitted from the star (which is negative, i.e. in the past). Since we are using units such that the speed of light = 1 , the time will be equal to negative distance to the star.

In GLSL code,

```
vec3 p=star_pos-ship_pos;
float r=length(p)
vec4 spacetime_pos_world=vec4(p, -r);
vec4 spacetime_pos_local=lorentz_matrix*spacetime_pos_world;
```

After performing the transformation, you obtain a new spacetime 4-vector. The xyz component you can directly use for rendering, as
```
gl_Position = projectionMatrix*vec4(spacetime_pos_local.xyz, 1);
```
The w component allows to easily compute Doppler coefficient (how much the wavelengths are blue or red shifted)
```
float df=spacetime_pos_local.w/spacetime_pos_world.w;
```
Note that the same number of wavelengths must fit into a spacetime interval, according to all observers. So the ratio of times gives us a change in wavelength of the light.

The Doppler effect posits a bit of a problem. It can not be simply applied to starlight's RGB values, as when starlight is blue shifted, infrared will be shifted into red. And we don't have infrared in our RGB.

Fortunately for us, starlight can be closely approximated as black body radiation. And black body radiation has a very convenient property: blue or red shifted black body radiation corresponds to black body radiation at a different temperature, with an intensity correction for the change in the solid angle at which we see it.

This correction is only necessary because we are rendering the star as a single pixel point - always one pixel in size. While the relativistic aberration would change the apparent size of the star. If we had been ray-tracing an object, fewer rays would've hit the object, reducing its intensity by the same factor.

So, we can compute the correct temperature and then compute star's color. The temperature is simply multiplied by the Doppler factor. The intensity correction is a bit more difficult; it can be derived from first principles, or taken from a paper by Montgomery Johnson and Edward Teller (the H-bomb guy), called "Intensity changes in the Doppler effect". The correction is a simple division by Doppler factor squared.

While we're at it, we also need to account for distance to the star, by dividing the brightness by distance squared.

The resulting code looks like this:
```
vec3 col = blackbody_radiation(temperature*df)*(star_brightness/(r2*df*df));
```
where blackbody_radiation function converts a temperature into an RGB value, and df is the Doppler factor.

## Lorentz Matrices

Above, I had referred a lot to "Lorentz matrices". What is a Lorentz matrix? It is a matrix describing space-time transformations (Lorentz transformations), similarly to how matrices are used to describe space transformations in computer graphics.

Lorentz transformations include any combinations of change of the reference frame (speed) and ordinary rotations and translations. Thus a full Lorentz transformation is represented with a 4x4 matrix and a translation four-vector.
Numeric precision and four velocities

As you accelerate in a spaceship, the speed approaches, but never reaches the speed of light. Additionally, it does not take a very long time until the speed becomes so close to the speed of light (1) that the numeric precision is insufficient for distinguishing it from 1.

![The ship is flying at 99.99999999999999999999999999999999999999999999999999999999949% of the speed of light.](https://github.com/Dmytry/space/blob/master/article_img_2.png?raw=true)

To work around this problem, I used four-velocities to represent ship velocity. A four velocity is ship's velocity as (x,y,z,1)*γ . Four velocities do not suffer as much from numerical precision issues, as they don't converge to 1, but instead continue growing without a limit as ship accelerates.

Another advantage of using four velocities is that velocity addition is considerably more straightforward (although you need to "renormalize" the four-velocity afterwards).

The numerical precision issues are of general interest, as they arise in all sorts of computations where an important value converges to a limit other than 1.
Displaying numbers that are very very close to 1

With double precision numbers, the maximum number of nines that would work correctly, would be around 16 at best. So how do I get a page full of nines above?

To be able to display greater speeds correctly, I used a trick. I calculated 1 - speed from γ as
```
function gamma_to_one_minus_beta(gamma){
  if(gamma<1E3){
    return 1-Math.sqrt(1-1/(gamma*gamma));
  }else{
    return 1/(2*gamma*gamma);
  }
}
```
and convert that value directly to the percentage:
```
function one_minus_x_percent_string(x){
  if(x>=0.009)return ((1-x)*100).toFixed(2);
  var nines=Math.floor(-Math.log10(x));
  var multiplier=Math.pow(10, nines);
  var last_2_digits=Math.floor(100-x*multiplier*100).toString();

  return "99."+"9".repeat(nines-2)+("0".repeat(2-last_2_digits.length))+last_2_digits;
}
```
## Cosmic microwave background radiation

One interesting effect that I wanted to show, was that if you were in a spaceship that is going fast enough, you would start seeing the cosmic microwave background radiation, as it gets blue shifted enough to be visible.

![A screenshot showing blue shifted cosmic microwave background radiation](https://github.com/Dmytry/space/blob/master/article_img_3.png?raw=true)

I reused parts of my stars shader, but appied them to an icosahedron to simulate a dome around the user. At the relevant blue shift, the aberration is so strong that the dome is effectively collapsed to a single point.
Limitations of float16 buffer

![A screenshot showing artifacts caused by denormal numbers](https://github.com/Dmytry/space/blob/master/article_img_4.png?raw=true)

I render my stars into a 16 bit precision floating point render buffer, prior to applying bloom effect. For it to work correctly over the entire immense brightness range encountered by the spaceship, I had to rescale output values accordingly (by applying a scaling in the shader prior to writing to the render buffer, and scaling again when rendering to the screen).

Another issue is that mobile platforms do not support denormal 16 bit floating numbers. There is a substantial hole in precision around 0.

![A screenshot showing no denormal number artifacts when using offset fix](https://github.com/Dmytry/space/blob/master/article_img_5.png?raw=true)

To work around this, instead of clearing my float16 render buffer to 0 (black), I clear it to lowest grey that still works correctly. Then, after bloom, when I render to the screen, I subtract that grey (appropriately correcting for the increase in brightness caused by the bloom effect). Without applying this trick, the rendering looks awful - instead of smoothly transitioning to black, there is an abrupt transition.

## Procedurally generating the starfield

I generate a starfield from a pre-computed grid block of stars, using a form of a hash function in GLSL.

I do that at several scales, to reproduce rare, very bright stars, and more common, less bright stars.

This does not really correspond to a true brightness-color distribution of real stars, and one of the next steps in development will be to build and use a table based on nearby stars catalog.