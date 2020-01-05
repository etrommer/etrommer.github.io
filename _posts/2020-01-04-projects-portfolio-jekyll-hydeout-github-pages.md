---
layout: post
title: A personal website with Jekyll and Github Pages
categories:
  - Projects
---

I've dabbled with Wordpress for my personal website before, but it always felt like too much of a hassle for the few things that my website needs to contain. All I really want is static text, a few pictures, code highlighting, preferably in a way that is pleasant enough to look at for extend periods of time. So why spend a lot of time and money on setting up and running databases, JavaScript and PHP? Since a work with git and Markdown most of the time anyway, [Jekyll](https://jekyllrb.com/) is pretty much perfect for what I need. After some very basic setup, it
provides a static website, just from Markdown documents and can publish through [Github Pages](https://pages.github.com/), simply by pushing to a Github Repository.

## Hydeout
After some looking around, I've settled on the excellent Jekyll theme [Hydeout](https://fongandrew.github.io/hydeout/) which is free under the MIT license, and to which I only had to make a few alterations to tailor it to my liking.

## Setting up Jekyll and Hydeout
Disclaimer: This is the first time ever that I'm doing things in the [Ruby](https://www.ruby-lang.org/en/) ecosystem, so this is how I got it to work, not necessarily how you _should_ get it to work. That said, so far it seems to be working well for me so far. Since I wanted to make some changes to the layout itself, I decided to not use Ruby's package manager _Bundler_ to handle the dependencies (including the theme itself) but instead install the required dependencies directly:

```bash
gem install jekyll jekyll-paginate jekyll-gist jekyll-feed jekyll-sitemap
```

with that, we can clone our local copy of the theme:

```bash
git clone https://fongandrew.github.io/hydeout/ && cd hydeout
```

I then removed the Gemfile so that Ruby doesn't get confused by missing dependencies and started serving the site locally.

```bash
rm Gemfile
jekyll serve
```

this starts serving the website in my browser under `localhost:4000`.

## Changes to Hydeout
Most of the changes were very basic
