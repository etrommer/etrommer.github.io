+++
title = "Moving my site from Jekyll to Zola"
+++

...because who doesn't love an occasional rewrite?
<!-- more -->
When I first set up my old website, I didn't fully understand what goes into a website that's hosted on Github Pages. I also wasn't entirely sure what the primary use case for it would be. Would I mainly use it as a blog? Should it be a documentation of past projects? Back then, I went with Github Pages' native static site generator [Jekyll](https://jekyllrb.com/). I ended up customizing the [Hydeout](https://github.com/fongandrew/hydeout) template, which
I was quite happy with.

With a couple of years passed, I'm changing the focus of my website to be primarily an online CV and portfolio with the option to blog in highly irregular intervals whenever there is something interesting. In the meantime, I had figured out that hosting a static site on Github Pages doesn't require you to use Jekyll. Running Jekyll locally also became a dependency nightmare, which is why I looked into other SSGs. I eventually stuck with [Zola](https://www.getzola.org) (because Rust!) and
have been pretty happy with the experience so far. It gets the job done nicely while it doesn't kill you with complexity. It's super fast and self-contained, which makes it really straigh-forward to set up on any system. Adding a few features that were present in [Hydeout](https://github.com/fongandrew/hydeout) but not in the Zola version only took a couple of hours.
