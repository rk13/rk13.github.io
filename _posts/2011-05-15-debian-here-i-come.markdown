---
layout: post
title:  "Debian, here I come"
date:   2011-05-15 02:00:28
---

I started to use Debian GNU/Linux more that 5 years ago for server-side. The small hosting environment I am currently playing with at my leisure runs mostly Lenny and Squeeze instances. About 3 years ago I have also decided to start using Ubuntu as my primary desktop operating system at home. The transition period was hard and painful sometimes, but now my laptop and PC are running Lucid and Karmic.

Being a hobby admin I am thankful to Debian for stability and quality of this rock-solid distribution. At the same time being an open-source compassionate, I am really impressed by the spirit of do-ocracy and a high level of commitment in the Debian community.

The idea to contribute to the project appeared some time ago. The development of Debian is open to all, and new users with the right skills or the willingness to learn are welcome to maintain orphaned packages, to develop new ones, and to provide user support. Being a software developer with Java experience I decided to try myself in a role of Java package maintainer.  While it is usually advised to start with fixing existing packages or to package something small and relatively easy to accumulate the experience first, I was rather ambitious and aimed to package [SQuirreL client][squirrelsql], that I am using daily at work and at home.

The work is still in progress and I wish I could spend more time on this, but I am quite seriously concerned about getting squirrel-sql, plugins and missing dependencies packaged. At this point I am very proud to announce that the first bits of my efforts have been [accepted][rsyntaxtextarea] to [Debian recently][debian-news]. Sure, it might take some time before all issues are resolved and other packages are accepted. Meanwhile, you may try it from [my ppa][ppa].

Apart from pure technical challenges non-official developers (like me) need to go through the s.c. sponsorship process in order to get their packages to the official Debian distribution,  and my big thanks to Niels Thykier and pkg-java team for helping me with this. I am also really impressed about the response  and the help I have received from the “upstream”, thanks Robert Manning (squirrel-sql) and Robert Futrell (rsyntaxtextarea).

--

http://www.squirrelsql.org/

http://packages.debian.org/source/sid/rsyntaxtextarea

http://www.debian.org/News/weekly/2011/08/#newcontributors

https://launchpad.net/~rk13/+archive/rk13-ppa

[squirrelsql]: http://www.squirrelsql.org/
[rsyntaxtextarea]: http://packages.debian.org/source/sid/rsyntaxtextarea
[debian-news]: http://www.debian.org/News/weekly/2011/08/#newcontributors
[ppa]: https://launchpad.net/~rk13/+archive/rk13-ppa

