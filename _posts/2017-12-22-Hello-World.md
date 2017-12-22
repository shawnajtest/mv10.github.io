---
layout: default
title: hello_world.asm
tags: personal
---

![hello world](/assets/2017/12-22-helloworld.png)

## hello, world
"Everybody should blog," said nobody, ever. And yet here we are. I intend for the blog to focus primarily on software-related posts, particularly C#, .NET, and Unity game development, with the occasional foray into weird stuff I'm doing at the office like IBM ODM business rules (I can't believe some people actually like using Eclipse). I suppose we can add web development to that list. I'm not especially fond of web development -- I consider HTML, CSS, and Javascript three trainwrecks that just never seem to end -- but it pays the bills better than most things I can do from the comfort of an air-conditioned office. Non-software topics are bound to show up, too. Electronics, AI, motorsports come to mind, though I've been out of the racing loop for a few years now. Heck, maybe even beer, guns, and small dogs. You never know.

<!--more-->

### http:<i></i>//about:jonmcguire

This post is basically a test-run, which is hosted as a GitHub personal pages site. This sort of content-management was largely inspired by [Marc Duiker's post](https://blog.marcduiker.nl/2015/10/06/moving-my-blog-i-love-github-and-markdown.html) detailing how he streamlined his own blogging process (and avoided the headaches of owning and operating a real CMS).

So what's this "mcguirev10" thing all about? For nearly 20 years, it has (mostly) been my username, sometimes abbreviated to "MV10". Back in October of 1999, I decided towing our big boat with a Dodge Durango was no fun, so I placed an order for a Dodge Ram 2500 with a V10. The Facebook guys were still in high school, and discussion forums were the social media of the day. I had a question about the truck, but someone else already had the username "mcguire" on the truck owner's forum, so I added the V10 and the rest is history. We also bought a Viper in 2001 and kept that for 16 years, so most people assume it's *that* V10, but the reality is my mudding days preceded my track days.

![going racing](/assets/2017/12-22-racing.jpg)

Anyway, as I mentioned, this post is really a test for sorting out the templates and styles and other browser-mandated arcana, but here's some 8086 assembly to spice things up before we leave.

{% highlight nasm linenos %}
```
DATA SEGMENT
     MSG DB "hello, world$"
ENDS
CODE SEGMENT  
    ASSUME DS:DATA CS:CODE
START:
      MOV AX,DATA
      MOV DS,AX
      MOV DX,OFFSET MSG       
      MOV AH,9H
      INT 21H
      MOV AH,4CH
      INT 21H      
END START
ENDS
```
{% endhighlight %}