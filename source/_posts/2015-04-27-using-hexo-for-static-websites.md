---
title: "Using Hexo for Static Websites"
date: 2015-04-27 07:56:35
tags:
    - JavaScript
    - NodeJS
---

##The Problem
I've been doing some work to setup some basic websites (and moving my blog off of blogger). I really wanted to take advantage of a [static site generator](https://www.staticgen.com/), and I wanted to utilize some of the best cheap hosting I could find (i.e. free). I know that I can point a custom domain to a [GitHub Pages](https://pages.github.com/) repository for nothing. GitHub Pages supports Jekyll as a static site generator - but I really didn't want to have to go down the road of setting up Jekyll on my Windows machine (it runs on Ruby and I've had very little luck with all the Ruby Windows installation tools out there). I'm not against Ruby, and I have a Mac partition on my machine - but it's not my primary working environment. **What I really want is a site generator that runs on both platforms, like on NodeJS.**

##Enter [Hexo](https://www.hexo.io)
I'll spare the intro to [Hexo](https://www.hexo.io) - but suffice it to say it's one of the more (if not most) popular NodeJS-based site generators out there. It's actively contributed to (from folks all over the world) - and I found it quite easy to grok once I got started. The [Docs](https://hexo.io/docs/) are pretty good, and I was able to pretty much completely move my blog and build a custom theme over the course of a couple evenings.

##Why Hexo?
There were a number of things that helped me land on Hexo. For the most part it came down to: 
* GitHub-Flavored Markdown support
* Official support for LESS
* [Permalink](https://hexo.io/docs/permalinks.html)-extensibility (so I could prevent broken links when migrating sites)
* Easily customizable themes
* [EJS](http://ejs.co/) support (more on that below)
* Supported deployment story with Git and GitHub

I mentioned EJS templating support. I realize that there are tons of other new and shiny templating languages out there - but one thing I really like about EJS is the fact that, as a developer, I can drop down and write raw JavaScript (the language EJS is written in) to get things done. A lot of other templating languages might be written in a different language but supported in Hexo (or other static site generators) by having a wrapper layer over them... which generally means there's not a good story for working with the bare language if need be. 

Overall, I'm pretty happy with Hexo and I hope I get some more chances to use it. I still primarily work in .NET and JavaScript - so I'm not making some big jump to web-design-only work, but this was a fun little project. You can check out the [source](https://github.com/ericmbarnard/ericmbarnard.github.io-hexo) of this blog and see what I did.