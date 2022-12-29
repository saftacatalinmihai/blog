---
layout: post
title: Sleep better with Raspberry Pi automated white noise
excerpt: Self hosted white noise
date: 2021-01-31
updatedDate: 2021-01-31
banner: rpi-white-noise.png
tags:
  - post
  - automation
---

### Theory

Having complete silence while you sleep 😴 may seem nice, but in fact can lead to trouble if the outside environment is not also completely quiet.

A cat may meow🐱, a car may honk 🚗, your upstairs neighbour can wake up for a late-night snack 🍟… All these sounds will seem louder if your room is completely quiet.

The human perception of sound is nonlinear. That’s why it’s measured in [decibels](https://en.wikipedia.org/wiki/Decibel). We describe a 10 times increase in volume as a 10 points increase in the decibel measure.

Also, the brain is very adept at filtering out constant noise. After a short acclimation period, the brain will consider the noise to be silence.\
They use this trick in horror movies. They place a constant white noise in the background and just before the [jump scare](https://en.wikipedia.org/wiki/Jump_scare), they quiet down that noise also, which makes the silence seem more silent 😈

That’s the logic behind using white noise for sleeping: we intentionally add a background noise for our brains to filter out.
Then, if there are outside noises are below the db level of our background white noise, we basically won’t hear them anymore and can sleep soundly without interruptions 😴.

### Implementation

Connect your Pi to some speakers

<div align="center">
    <img alt="raspberry-pi-white-noise" title="Raspberry py white noise" src="/rpi-white-noise.png">
</div>

Write a script: `/home/pi/bin/whitenose.sh`
It will call [omxplayer](https://www.raspberrypi.org/documentation/raspbian/applications/omxplayer.md) to start your preferred white noise sound.
I used a free sample Airplane Cabin noise I found on the internet (10 hours long mp3, 1.4 GB) and added it to the /media directory of the Pi.

```shell
#!/usr/bin/env bash
nohup omxplayer /media/Airpane_Cabin_White_Noise.mp3 &> ~/whitenoise.log
```

Don’t forget to make it executable:

`chmod +x /home/pi/bin/whitenose.sh`

Then install the script in the Raspberry’s [cron](https://en.wikipedia.org/wiki/Cron) at your desired hour to go to sleep, in my case it’s [23:00](https://crontab.guru/#0_23_*).

`0 23 * * * /home/pi/bin/whitenoise.sh &`

That’s it… Every evening at 23:00 the sound will start.

No need for white noise apps, no subscriptions, no asking Google Home to start white noise every evening, no internet connection required.
Just a cheap Raspbery Pi, some old speakers and a bit of linux knowledge.

<div align="center">
    <img alt="huzzah" title="Huzzah!" src="/huzzah!.gif">
</div>

Of course this can work on any old Linux machine you have lying around.

Hope this helps 😀
Sleep well!

Edit:
Discussion link on Reddit: [selfhosted](https://www.reddit.com/r/selfhosted/comments/l9bs85/sleeping_better_with_raspberry_pi_automated_white/)
