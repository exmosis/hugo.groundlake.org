---
title: New-hu-go-go
date: 2021-11-29
---

Obligatory "FRIST PSOT" post. I have been largely following [this tutorial](https://www.freecodecamp.org/news/how-to-build-a-blog-using-a-static-site-generator-and-a-cdn/) so far, and things have been mostly fairly smooth. My own approach has been:

* Install hugo via snap, as I'm on Ubuntu 21.10 (`snap install hugo --channel=extended`)
* Run through the new site setup to create a new directory
* Set up a new github repo and clone it
* Copy the new site files into the git repo and push
* Clone the [beautifulhugo](https://themes.gohugo.io/themes/beautifulhugo/) theme as a submodule, as suggested (glad I had all that submodule wrangling before now, aren't I, eh, eh?)
* Push
* Sign into Netlify via Github, giving it just access to the one repo
* Set up new site, roughly in line with the tutorial steps
* Set up hugo.groundlake.org over at [Mythic Beasts](https://www.mythic-beasts.com/) which is hosting the groundlake.org domain, in line with Netlify's instructions
* Verify the DNS - this was fast, but not entirely sure if it has worked. HTTPS seems to work when I access the site, but adding SSL seems to report "missing certificate", and I got an email from Netlify about mixed secure/non-secure content, which the browser doesn't report.
* Figure out how to post this content. I've set up a '2021' directory within my 'posts' directory, just in case I end up using this for years and years. Running 'hugo server' seems to see it fine.

(Actually, I originally added the theme as a clone before the parent was pushed, but undid that once I realised Netlify couldn't see it.)

About a minute after posting this, it appears on the site - hooray!

So my initial issue is that the netlify-generated site looks quite different to the one hosted locally. This ain't right:

![Screenshot of badly formatted blog](/img/post/Screenshot_20211129_125159.png)

After adding a quick 'netlify.toml' file as per [Hugo's instructions](https://gohugo.io/hosting-and-deployment/hosting-on-netlify/) to set the same hugo version, everything is looking pretty good though.
