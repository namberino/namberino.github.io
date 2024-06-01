---
title: "Fire! But cold"
date: 2024-04-18T20:57:47+07:00
tags:
  - science
  - physics
  - electrical
author: "Nam Nguyen"
description: "Exploring the world of cold fire (which, yes, you can actually touch it)"
---

I recently came across some experiment on youtube about something called *"cold fire"*. It sound contradictory, right? How can fire, something that is inherently hot, be cold? Well, that is what we are going to explore in this blog post.

# What is fire?

To understand fire, we need to understand **plasma**. It is the 4th state of matter, right after gas. When a gas gets extremely hot, the electrons inside a substance have enough energy to detach from their atom and move freely. This makes the atoms become highly charged due to the lack of electrons and they form a collection of charged particles. 

Fire can be defined **partial plasma** because fire's ionization is low due to the very low percentage of ionized atoms compared to the total number of atoms in the gas. Even though it is considered a low grade plasma compared to something like for example lightning, it is still extremely hot. 

Something that is important to understand is that a plasma has 2 sets of temperature:
- Electron temperature
- Ion/Atom temperature

Because of the way fire is released (*A burst of energy*), the atoms get very hot as the reaction happens and the electrons move around. This is actually what happens in most plasmas. If you put more energy into the plasma, the temperature of the atoms rise substantially. This is called hot plasma (or its scientific name "Local Thermal Equilibrium Plasma" because the temperature of atoms and electrons are equal) 

{{< image src="/img/cold-fire/match-fire.jpg" alt="Plasma on match" position="center" style="padding: 10px" >}}

However, this isn't the only option. If you somehow apply energy in someway that just heats up the electron and keep the atoms cold, you'd have a plasma that has an electron temperature of around *3 - 15* thousand degrees. This is called cold plasma (or its scientific name "Non Thermal Equilibrium Plasma" because the atoms are at a much lower temperature compared to the electrons, which brings down the average temperature of the plasma). This would actually feel cold because tiny electrons with very little mass compared to atoms don't have the momentum to transfer heat to a large object.

So you can actually touch a cold plasma torch without hurting yourself.

{{< image src="/img/cold-fire/cold-plasma-finger.jpg" alt="Finger touching cold plasma" position="center" style="padding: 10px" >}}

# The making of cold plasma

Now that we got the basic theory of plasma and cold plasma down, how can we make cold plasma?

Anyone who's ever played with high voltage (any fellow electrical engineers and enthusiasts out there) knows that high voltage sources allows you to pull a long stream of plasma as soon as the wires get close enough. If you want to see this in action, just watch [this Electroboom video](https://youtu.be/m7VP36diOKY?si=Bape72WkVFGqrr1b&t=132) 

This happens because the electrons in the wires have enough energy to jump out of the wire over a fairly long distance to the common ground to release their energy. As the wires get closer, some electrons jump over the short gap, this will ionize the air molecules around the wires, making more electrons jump over. This also make the air more conductive as the air's electrons are knocked out, which means the electrons has a conductive path to flow through. The more electrons flow, the more ionized the air becomes, the hotter everything around this becomes. 

So we can take advantage of this phenomenon to create a stable stream of plasma. We can make this more stable by increasing the voltage but that's pretty costly, so we increase the frequency of the voltage to create a stable plasma arc (the frequency should be in the low radio ranges).

A typical microwave transformer running from city power lines is around *60Hz* or 60 times per second. The distance that the electrons can jump is proportional to the voltage and 60Hz can create longer wait time between the peaks, where the electrons have the most energy and can jump the farthest. By increasing the frequency, there's more frequent voltage peaks, more time for the electrons to jump with the highest energy.

By doing this, we all so create something called a *"Far field effect"*. If we increase the frequency of to low radio ranges, the wire will start to radiate radio waves which can energize the electrons. And we know that more energy means more jump distance. 

We can utilize this by directing the electrons by directing them through an insulating tube through a stream of easily ionizable gas like helium or argon. The electrons will ionize the gas, turning it into plasma and the high frequency will help keep this energized. 

Because the gas atom isn't exposed to enough energy to heat them up in the short time they are in the tube, the electrons will absorb most of the energy, making the plasma cold plasma.

# But why cold plasma?

There's a couple reasons why we need cold plasma:
- The cold plasma is full of highly reactive charged particles, which can destroy microorganisms very quickly. So this can be used for sterilization without harming the thing it's sterilizing. 
- The reactive charged particles can be used inside chemistry like seperating a chemical compound.
- It's just so cool. I mean come on, it's like magic in real life.

# Making cold plasma yourself

> **Disclaimer**: Working with high voltage is **VERY** dangerous. If you don't know what you're doing or you don't have much experience with this, **DO NOT DO THIS**. If you are going to make cold plasma, make sure to take proper precautions.

If you have a high voltage, high frequency generator, some insulating tubes and a tank of easily ionizable gas, you can make cold plasma yourself. You can check out this tutorial from the *Plasma Channel* if you want to make one:

{{< youtube wOV8kliF4eo >}}
