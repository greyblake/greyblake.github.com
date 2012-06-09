---
layout: post
title: "Beep alarm"
date: "2011-01-23 23:24:39"
comments: true
categories: beep alarm Linux bash
---

## Intro 

Someone would say I am brainsick... So may be I am.

By some reasons I had no my loudspeakers and an alarm on my mobile never wake me up.
So I decided to create an alarm based on `beep` Unix tool. This tool allows you to play sound with specified frequency and delay on your speaker.

Some times after I decided that simple "Be-e-e-e-p" sound is borring and I created a player.
It can play tunes defined in format for old Nokia mobiles.
Have you ever type something like this:

    8e2 8d2 8d2 8c2 8b1 8a1 8b1 8c2 8f1 8e2 8d2 8- 8- 8d2 8c2 8b1 

...to add new ringtone for your mobile?


## Getting and usage

You should clone my [repository from github](http://github.com/greyblake/beep-alarm):

    git clone git://github.com/greyblake/beep-alarm.git

And to try it just type:

    ./beep-alarm/bin/beep-alarm

You should listen a sound of russian song "Kalinka". Unfortunately the only way to stop it is to send INT signal with pressing `Ctrl-C`.

With this tool you can specify time at what you want a tune to play. Next example make it play at 8:00:

    beep-alarm 8:00

It has 7 bultin melodies(to see them use `--help`), so you can choose another one with `-m` option:

    # play Metallica - Unforgiven
    beep-alarm -m 3

You can even change tempo of tune(default is 120)!

    beep-alarm -t 200

And even play a tune in different keynote:
    
    # Down for 2 semitones
    beep-alarm -o -2

To get more information and example use `--help` option.

## So...

I hope you'll enjoy it and will never oversleep your work:)

