---
layout: post
title: A personal website with Jekyll and Github Pages
categories:
  - Projects
---

I've dabbled with Wordpress for my personal website before, but it always felt like too much of a hassle for the few things that my website needs to contain. All I really want is static text, a few pictures, code highlighting, preferably in a way that is pleasant enough to look at for extend periods of time. So why spend a lot of time and money on setting up and running databases, JavaScript and PHP? Since a work with git and Markdown most of the time anyway, [Jekyll](https://jekyllrb.com/) is pretty much perfect for what I need. After some very basic setup, it provides a static website, just from Markdown documents and can publish through [Github Pages](https://pages.github.com/), simply by pushing to a Github Repository.

## Hydeout
After some looking around, I've settled on the excellent Jekyll theme [Hydeout](https://fongandrew.github.io/hydeout/) which is free under the MIT license, and to which I only had to make a few alterations to tailor it to my liking.

## Setting up Jekyll and Hydeout
I wanted to make a few more alterations to the layout than were provided through the options in the SASS file as described in the [Hydeout Readme](https://github.com/fongandrew/hydeout#customization). I decided to not use Ruby's dependency manager _Bundler_. We will therefore install the plugins that Github provides directly via Gem:

```bash
gem install github-pages
```

with that, we can clone our local copy of the theme:

```bash
git clone https://fongandrew.github.io/hydeout/ && cd hydeout
```

We then remove the Gemfile so that Ruby doesn't get confused by missing dependencies. After that, we can start looking at the site locally and make changes to it.

```bash
rm Gemfile
jekyll serve
```

this starts serving the website in the browser under `localhost:4000`.

## Configuration
The first step is to adapt the Jekyll Config to your needs. Here's what mine looks like:
```yaml
# Dependencies
markdown:         kramdown
highlighter:      rouge

# Setup 
title:            'Elias Trommer'
tagline:          'Tech Blog'
description:      'doing digital stuff'
url:              https://etrommer.github.io

author:
    name:         'Elias Trommer'

social-network-links:
    github:       https://github.com/jadeaffenjaeger
    linkedin:     https://www.linkedin.com/in/elias-trommer-58a085138

# How many entries per page?
paginate: 10

# Used plugins go here
plugins:
    - jekyll-feed
    - jekyll-gist
    - jekyll-paginate
    - jekyll-sitemap

# Sidebar link settings
sidebar_home_link:  true
```

Make sure to remove the line `baseurl: '/hydeout'` if you are running on Github Pages, as otherwise your HTML paths will not be generated correctly. You might have noticed that the `social-network-links` section is not present in the original config. It contains the links to my social network profiles, with which the icons in the sidebar are generated. I'll describe below how to add them to the layout.

## CSS

To modify the default SCSS, you'll need to add an additional SCSS file that gets read first under `assets/css` as described in the [Hydeout Costumization](https://github.com/fongandrew/hydeout#customization) section of the Readme. I've chosen to go with blueish color as the main color and made the font a bit smaller. 

```scss
---
# Use a comment to ensure Jekyll reads the file to be transformed into CSS later
# only main files contain this front matter, not partials.
---

$sidebar-bg-color: #23373B;   # color of sidebar
$sidebar-fg-color: #e0e0e0;   # color of sidebar font
$body-bg: #f9f9f9;            # main background color
$link-color: #b33f62;
$code-color: #707a8b;

$sidebar-sticky: false;

$large-font-size: 1.1rem;

$sidebar-width: 24rem;

@import "solarized.css";
@import "hydeout";
```

I've also changed the color scheme of the code highlighting to the [Solarized Dark](https://gist.github.com/nicolashery/5765395) color scheme which nicely matches the overall teal look of the rest of the website. To do this, find a CSS colorscheme for Pygments and put it in the folder `assets/css`. The line `@import "solarized.css";` then imports the file by that name. To override the defaults of Hydeout, you'll also have to comment out the line `@import "hydeout/syntax";` in the file `_sass/hydeout.scss`. This will make sure that the default code highlighting configuration will not be included.

## Social network icons

I've also added a few social icons in the sidebar. To do the same thing, find the icon of your preferred social network as an SVG file and place it under `_includes/svg`. Then add an entry like this to the file `_includes/custom_icon_links.html`:

```html
{% raw %}
{% if site.social-network-links.github %}
<a id="github-link"
   class="icon"
   title="Github" aria-label="Github"
   target="_blank"
   href="{{ site.social-network-links.github }}">
  {% include svg/github.svg %}
</a>
{% endif %}
{% endraw %}
```

This will check if the `config.yml` contains a link to Github and if so, will generate a social network icon in the sidebar. If the layout seems off, check whether your SVG icon has width and height informations embedded in it. If it does, you'll need to remove them by opening the SVG in a text editor. Find the keywords `width` and `height` and simply remove them from the file. The icon should now be rendered to the correct size.

## Publishing your website

So far, your website only lives on your computer. To put it on the internet, you'll need to go to [Github](https://github.com) and create a new repository. The name of your new repository *must* follow the convention `<your username>.github.io`. Also make sure that you **uncheck** the option "Initialize this repository with a README" when creating the repository. Once you're done, click the "Clone or download" button in your repository and copy the URL. You can publish your website with the command

```bash
git remote set-url origin <your-repository-url>
git push
```

If everything worked out, your website should now be available under `<your username>.github.io`!