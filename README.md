# Relativistic spaceflight simulator

Special thanks to Mr.doob for three.js (WebGL wrapper and scene graph renderer) and Silvio Paganini for help with adapting my bloom effect to three.js

This is a relativistic spaceflight simulator which shows what it would look like if you were on a 
spaceship traveling close to the speed of light. 
It is loosely based on my original code from 2012 implementing relativistic effects in my game The Polynomial. 

It simulates relativistic boost (for spaceship motion), relativistic aberration and Doppler effect (both frequency and intensity). It also simulates cosmic microwave background radiation.

The view is displayed as 1 year = 1 second timelapse. 

The code is currently a bit of a mess and needs a cleanup.

The general development philosophy is to make something that works now and will still work in 10 years. Consequently, the javascript is written directly rather than using a translation layer, node.js is not used, etc.

Copyright: 2020 Dmitry Lavrov.

License: Attribution-ShareAlike CC BY-SA https://creativecommons.org/licenses/by-sa/4.0/legalcode for the index.html
