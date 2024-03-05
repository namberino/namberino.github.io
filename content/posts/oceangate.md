---
title: "The pressure of OceanGate's Titan"
date: 2024-02-19T12:27:06+07:00
tags:
  - physics
  - math
author: "Nam Nguyen"
description: "The physics of the OceanGate's Titan incident"
math: true
---

I was bored so I went down the rabbit hole of engineering disasters. One disaster that caught my eyes was the OceanGate's Titan incident. It caught my eyes because I was curious about how much pressure was applied onto the submarine at that depth, so I grab my calculator and got to work.

The most dangerous thing about deep ocean is probably the hydrostatic pressure. This pressure is proportional to the depth measured from the surface of the water body. The weight of the body of fluid gradually increases.

This hydrostatic pressure is calculated with the following formula (assuming the pressure is measured over a *$1m^2$* liquid block):

$$
P = \rho g h
$$

Where:

*$P$*: Pressure of the liquid ($N/m^2$)

*$\rho$*: Density of the liquid ($kg/m^3$)

*$g$*: Acceleration due to gravity ($9.8 m/s^2$)

*$h$*: The height of the liquid column

{{< image src="/img/pressure.png" alt="pressure diagram" position="center" style="padding: 10px" >}}

First, we can calculate the pressure that a normal human may experience on the surface and then multiply it with the depth of the OceanGate's Titan.

We already know some variable:
- The pressure ($P$) that we usually experience is $1atm$ or $100,000 N/m^2$
- The density of water is $1,000 kg/m^3$
- $g$ is $9.8 m/s^2$ but we'll round it up to $10 m/s^2$

So we can calculate that:

$$
\frac{100,000 N/m^2}{1,000 kg/m^3 \cdot 10 m/s^2} = 10m
$$

So the pressure increases by $1atm$ for every $10m$ of depth we go down. And since OceanGate's Titan was around $4,000m$ deep when it imploded, we can calculate that:

$$
\frac{4,000m}{10m} = 400atm
$$

So the OceanGate's Titan was experiencing around $400atm$ of pressure when it imploded. 

It's a bit hard to put $400atm$ into perspective. So for reference, here's a video of a can exploding under only $1atm$ of pressure:

{{< youtube atsgIvOUFhA >}}
