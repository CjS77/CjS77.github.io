---
layout: post
title:  "How I set up this blog"
date:   2015-04-01
categories: jekyll blog
---

# Setup

 - Pick a theme from [Jekyll themes](http://jekyllthemes.org/)
 - I went with the [Clean Blog](http://jekyllthemes.org/themes/clean-blog/) theme.
   - Now you can clone the code from GitHub, but I elected to just download the zip file and extract it to my
     repository folder. The code contains a full working Jekyll site that is based on Bootstrap and so
     *is responsive*.

## Configuration

The first thing to do is modify the settings in `_config.yml`. If you're using github-pages, you will need to 
set the `base_url` setting to the project name. 

When running the blog locally from jekyll, remember to override `base_url` from the command line.

You also need to run `npm install --dev` to enable the Grunt tasks that have kindly been set up for you.

The `less` task didn't work with my version of `grunt-less-contrib`. Removing the `cleancss: true` option does the trick
(presumably there's a plugin I'm missing somewhere).

## Customisation

The next thing to do is tweak the default settings of the Clean Blog theme:

### Images

  - Replace the stock images for the *About* and *Home* pages in `img/about-bg.jpg` and `img/home-bg.jpg`. These
    images are 1900 x 492. Use Gimp to create replacements and use a high compression factor to keep the image
    sizes small (~40kB).
    
### Fonts and aesthetics

The default fonts look fine, but I want to use my company font for headings. The sans-serif font for the posts is fine.

{:note: .note}

<div class="note" markdown='1'>
##### Don't edit the clean-blog.css file. It's gets overwritten

Edit the less files in the  `less` folder.  And remember to run `grunt less` after any changes 
</div>


In `mixins.less` I changed the following:

The default sans-serif font
  
     .sans-serif () {
     	font-family: Raleway, Helvetica, Arial, sans-serif;
     }
     
     .brand-font() {
         font-family: Syncopate, Raleway, Helvetica, Arial, sans-serif;
     }
  
The nice *notes* CSS mixin is also added:
  
    .box-shadow(@x: 0; @y: 0; @blur: 1px; @color: #0001) {
        -webkit-box-shadow: @arguments;
        -moz-box-shadow: @arguments;
        box-shadow: @arguments;
    }

In `clean-blog.less`
  - `h1` font uses the `brand-font` mixin. The default weight is also 400, rather than 800.
  - `body` font family is `serif` 
  - `.intro-header .post-heading .meta` should be `sans-serif` mixin call.
  
In `_includes/head.html` I changed the fonts that are loaded to `Syncopate` and `Raleway`.  

### "Plugins"

I really like the Notes CSS on the [Jekyll](http://jekyllrb.com) website. Cloning this behavior was a simple as 
setting up the .less file and `@importing` it into the `clean-blog.less` file.

However, jekyll doesn't import the `.less` files automatically, like it does with Sass, so you have to run `grunt less`
to incorporate any changes to the CSS.

The Fonts Awesome file is a set of icons in pure css. To speed things up, I downloaded a local copy into the CSS folder
and updated `head.html` accordingly.


### Posts

I replaced the default `about.html` file with `about.md`. I prefer using markdown and Jekyll will automatically
generate HTML from it.
