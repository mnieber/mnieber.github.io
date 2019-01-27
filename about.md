---
layout: page
title: About
permalink: /about/
---

Welcome to my blog. When it comes to programming, I like to call myself a pragmatic purist and hopefully this is reflected in my blog.

I'm pragmatic in the sense that I love simple solutions (sometimes even simplistic ones, as long as it gets the job done and can be upgraded later if necessary). For example, in my Dodo Commands tool the user can overlay the configuration file with layers of different flavours, such as a 'debug/on', 'debug/off' and 'debug/logging'. Calling 'dodo layer debug on' tells the system to overlay the configuration with the 'debug.on.yaml' file (and none of the other 'debug' flavours). The simplicity comes from inferring the set of possible layers and their flavours from the set of existing layer filenames. This is just a small example of how a simple and robust mechanism can offer a lot of expressivity, and I love that. Ever since, I have solved various Dodo Commands use-cases by adding a few switchable configuration layers.

I'm a purist in the sense I that try have a clear sense of what abstract concepts represent. For example, if a function deletes a user account unless it's locked then I would call it `deleteUserAccountIfNotLocked`. And if a function has a callback argument called 'setHighlight' then I make sure that this callback doesn't do anything unrelated to setting a highlight. This also applies at the system level: each part of the design has a specific purpose, and you are tempted to abuse this purpose (for example: allow the layout function to change the background colour of an item) then you should probably rethink the design or look for a different solution to the problem at hand. Very often, by sticking to the constraints of the design, you find a better solution than the one you initially thought of.

Apart from programming, I'm interested in buddhist meditation, swing dancing, listening to music of all kinds and playing the piano, guitar and surdo (the big drum in samba music).