---
id: 2
title: Light control using computer vision
excerpt: Light control using SCADA, soft computing and CV-based algorithms
thumbnail: '/assets/img/portfolio/thumbnails/light_control.gif'
---

# {{$frontmatter.title}}

This was a small side project I used as an introduction to my PhD field of
research. The idea was to use a SCADA system as an interface for control of the
lights and develop an intelligent agent capable of properly controlling them
using SCADA's interface. The result was an agent that connects to the SCADA
and, using a webcam, determines whether the lights in the room should be
amplified or lowered. User of the system could also affect the preferred
illumination levels using hand gestures. This was the result:

<iframe width="560" height="315"
        src="https://www.youtube.com/embed/IprXwcsyk9Q"
        frameborder="0" allow="autoplay; encrypted-media"
        allowfullscreen></iframe>

To clarify, Bri is the measured brightness of the current frame, Pref is the
preferred brightness of the current frame and Req is the approximated light
increase/decrease percentage.

## How it works

### Lights

In my apartment I use Phillips Hue light bulbs. The light bulbs connect to a
device called the Hue bridge. The bridge offers a REST interface that various
applications can use to control the light bulbs - turn them on/off, dim them or
even change colors (depending on what the lightbulb supports).

### SCADA

For the SCADA system I used an early prototype of Končar's new SCADA system. I
can't really go into details of the implementation here as the software is
commercial and I'm contracted under an NDA. What I can say is that I used it to
connect to the Hue bridge service (with some tweaks) and it provided an
interface that the intelligent agent used to increase or decrease the lights.

### Intelligent agent

The workflow of the intelligent agent was such that every second it would
connect to the webcam, get the current picture and decide whether it should
increase or decrease the lights (or keep them at the same level). This workflow
was implemented through a feedback loop, where the agent in every iteration had
its previous state and the current image, result of the iteration would be
delta indicating the change in lightning level and the update of the agent's
state. The state contained a series of previous images and a variable that
represents the currently preferred brightness. The analysis of the current
image and calculated result can be split into three categories: brightness
calculation, face detection, gesture recognition and fuzzy inference.

#### Brightness calculation

This one started out simple, but I managed to complicate it in order to get
better results. Initially, I would simply just take every pixel of the current
image in greyscale, average the values over the picture and use this as the
current brightness. This proved to be a bit problematic because if someone got
into the picture that was, for instance, wearing a black shirt, the agent would
think that the room got darker. In order to counter this, I've made the agent
store the images it gets into its state (and delete old ones after some time).
Images from the previous iterations would give me insight to temporal changes
in the picture - enabling it do detect which pixels change brightness rapidly.
The pixels that change brightness rapidly are more probable to be moving
objects, making them less relevant in the brightness calculation, giving
advantage to pixels that represent, for instance, walls. Brightness is then
calculated as a weighted average where weights are these relevancies. This
approach is still not perfect, but it works better than the basic one.

The following video demonstrates how this calculation works in practice, the
white pixels are more relevant then the grey or black ones:

<iframe width="560" height="315"
        src="https://www.youtube.com/embed/i8Yhj5-mtTo"
        frameborder="0" allow="autoplay; encrypted-media"
        allowfullscreen></iframe>

#### Face detection

One of the conclusions the agent can draw from the image is whether any people
are in the scene. If there are people, it should increase the preferred
brightness. For this, openCV's
[Haar cascading features model](https://github.com/opencv/opencv/blob/master/data/haarcascades/haarcascade_frontalface_alt.xml)
was used. This model proved to work well out of the box, so no further
alterations were required to solve this problem.

#### Gesture recognition

This is where things got interesting. I wanted to be able to be set preferred
brightness using hand gestures, open fist meaning "increase light", closed fist
meaning "reduce light". I started looking up tutorials how to detect human skin
and separate hands from backgrounds in the picture. I learned about different
image filtration methods and creating contours and convex hulls.

I was a bit inpatient and wanted results fast, so I ditched this approach
early, and decided to use a CNN. I bypassed the changing backgrounds problem by
fixing an image area and only performing gesture recognition there (red
rectangle in the first video). Since the leftmost pixel columns and upper rows
were rarely going to be used (hand doesn't go there often), I used these pixels
to define what color is the background, turning my refrigerator into an
improvised greenscreen. This made the hand detection much more efficient
(unless a refrigerator-colored alien was using the system). After that, a CNN
was trained on different images of my opened or closed fist, which gave
satisfying gesture detections.

This was used to update the currently preferred brightness - open fist
increases it, closed decreases. This can be seen on the first video written
under the "Pref" field.

#### Fuzzy inference

With the last three calculations the agent has everything it needs in order to
decide how current lightning level should change. What it needs is the
implementation that takes these values and calculates the delta. I used fuzzy
logic here with two simple rules:

  * If the room is too dark, increase light
  * If the room is too light, decrease light

Whether the room is too dark or too light is decided using the preferred
brightness. I modeled these variables as a sigmoid whose offset is determined
by the preferred brightness. Increase and decrease light are modeled as
sigmoids as well. The following figures demonstrate values of these variables
(if current preferred brightness is 20):

![Dark and light sets](./images/input_sets.svg)

For implications in the individual rules, I simply used the minimum function
and for rule result accumulation I used maximum. So if the camera measured that
the room was too dark, that would manifest with the following rule "firings":

![Fired rules](./images/fired_rules.svg)

In the end, I used center of area to determine a concrete, single value as an
output, which in above example would accumulate to 0.35, indicating an increase
in lightning by approximately 35%.

## Conclusion

This was a useful project mostly because it taught me new things about AI
integration in SCADA systems, gesture recognition and tools from openCV in 
general (this was my first practical use of it). It was also a nice revision of
CNNs in Tensorflow and implementations of fuzzy inference systems. It served
as a nice, fun introduction to my field of research for my PhD.

Practically, however, I don't see too much application for this system, mostly
because it requires a camera in order to control lighting. This itself is a
somewhat flawed approach because a number of things could go wrong:

  * If a person is not right in the scene, the camera won't detect it and
    increase the light, the room isn't completely covered.
  * The camera needs to see a face in order to increase the light, which means
    it needs to be able to find a face in the dark first - which puts us in a
    chicken-or-the-egg kind of scenario.

The camera approach is definitely not perfect, but it could make sense to use
it combined with some other methods, i.e. vocal control or a GUI over which the
preferred brightness can be changed.
