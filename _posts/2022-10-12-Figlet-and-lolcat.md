---
layout: post
title: Test all fonts in Figlet
excerpt: Using Figlet and Lolcat
tags: figlet lolcat 
---

<div class="header">
<img src="/images/figlet.gif" width="100%">
</div>

On Mac, using Homebrew

Install [figlet](http://www.figlet.org/)
```
brew install figlet
```

Install [lolcat](https://github.com/busyloop/lolcat) because why not?
```
brew install lolcat
```

Find out your font directory using `brew list figlet`, then based on the installed version (mine was 2.2.5) do:

```
for file in /opt/homebrew/Cellar/figlet/2.2.5/share/figlet/fonts/*.flf;\
do echo "\n\n$file"; figlet -c -w 150 -f "$file" Hello World |\
lolcat; done
```