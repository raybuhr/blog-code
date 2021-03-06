---
title: "Redesign"
date: 2017-05-04
draft: false
---

My previous blog design, as last blog post, used Rmarkdown (.Rmd) to
create and build out the site. Rmarkdown is a really powerful way to
communicate using the tools that R provides, but it is really slow and
painful to rebuild every post each time you make a change to the site.
Also, while you can always add custom css styling to your site and your
Rmarkdown files, I found that it was difficult to not spend countless
hours doing so and still end up with something I really liked. Lastly,
Rmarkdown websites do not allow for tree based directories, i.e. every
post has to be in a flat project directory. This makes it hard to scale
out the blog as you add more posts. Since I had in mind keeping this
around for a good long while, I wanted to incorporate an automatic
filing/archive system based on year/month routing.

Hugo
----

First off, [hugo](https://gohugo.io/) is a static website engine written
in Go. More importantly, it makes compiling blogs posts written in
markdown into rich, feature filled websites really fast and easy. If you
just want to write blog posts without using R to create tables and
charts or doing any computation, I highly recommend trying out hugo.

Blogdown
--------

The good people at RStudio, who created and maintain the Rmarkdown
library, also felt the same pain points I did using it to create a
static website (static means no server interaction necessary, but
includes interactivity through the browser via javascript). Their answer
seems to be to use hugo to build and render the website, but use
Rmarkdown to generate the html for individual pages and/or blog posts.
To do this, the have begun development of the library `blogdown`,
available [via github](https://github.com/rstudio/blogdown). In
addition, there is currently ***a whole book*** being written on how to
effectively use [blogdown](https://bookdown.org/yihui/blogdown/). From
my experience getting this redesign up and running, I totally understand
the need for such detailed documentation. Blogdown + hugo + Rmarkdown is
a really fantastic tool, but combining all three together was much more
challenging than I anticipated.

Lessons Learned
---------------

In order to remind future me (since I’m not great at creating posts in a
timely manner) and maybe offer some help to others looking to use
blogdown, here are some lessons I learned during the redesign of my blog
using the aforementioned technology.

**Create a blog project directory.**

-   You will want to iterate, **a lot**.
-   You may want to compare different versions of your website at the
    same time.
-   Having a folder where you can keep the different versions and tests
    makes your process (i.e. development cycle) a lot easier.
-   When you find a test that you start to like, promoting it to your
    hosting platform becomes much easier.

**Try out hugo by itself first.**

-   Hugo is an awesome tool and really simple to get started with, but
    does not assume you will be using other tooling (like RStudio,
    Rmarkdown, etc.) to build your site.
-   Getting the hang of hugo only takes a couple hours.
-   Getting proficient at tweaking your site until you are finally happy
    with it takes many hours.

**Try out blogdown by itself first.**

-   Blogdown is a just light hugo wrapper for R.
-   Blogdown assumes you have already learned how to use hugo to a
    minimal degree (the couple hours mentiond I just mentioned).
-   Blogdown works best *after* you have already decided which theme you
    want to use.
-   Delete the `config.yaml` file blogdown created. Hugo uses
    `config.toml` by default and doesn’t like having multiple config
    files.
-   Play with the `new_site()` parameters.
    -   For example, the `sample` argument will create sample content
        for you and the `theme_example` will create content based on
        whether the theme you pulled in has an `example_site` directory
        (most do).
    -   I recommend setting both to `TRUE` while you are testing out the
        look/feel of themes, but using neither when you decide on which
        you want to use.

**Try lots of themes**

-   Read the documention of the theme you want to use in detail.
    -   I tried nearly a dozen themes and they all do things similarly,
        but often times in slightly different ways.
-   The `config.toml`, like the `_site.yml` file used for Rmarkdown
    websites, controls how the website will build. Each theme has
    recommendations and tweaks to the basic config file so they all end
    up being a little different.
-   I recommend using a theme that supports external plugins like Google
    Analytics and Dsiqus. You can always add custom javascript to your
    pages, but it’s a lot easier to put your account into the
    `config.toml` and be done with it.

**Enjoy (or deal with) searching for help**

-   Understand the sometimes you may need to dig into front-end web
    stuff a bit to figure out why some complicated thing you are trying
    to do isn’t working.
-   Blogdown is still under active development and has some kinks to
    work out.
    -   For example, I learned that using the `datatables` (DT) library
        in R installs version of jquery and thus you run into javascript
        dependncy issues with hugo. The recommended way to solve this
        currently is to just create that element of your post as a
        separate page and embed it in an iframe (lame).

Using Github Pages to Host
--------------------------

If you can’t already tell, I host my blog using [Github
pages](https://pages.github.com/). Hugo, and thus blogdown, is not fully
supported by Github pages (…yet???). Instead, you’ll need to figure out
a way to only include the `/public` directory in your github pages
repository. There are a couple ways you can do this.

1.  Only copy the `/public` directory to your github repo and include an
    empty file named `.nojekyll` to tell github not to use Jekyll and to
    build your website.
2.  Create a branch just for displaying the website to the public using
    the reserved name `gh-pages` **AND** use the `git subtree push`
    command to only send the `/public` directory to that branch.

I opted for method 1 for now because it is a lot simpler, but I will
eventually move to method 2 so I can have all my code in Github.

More resources on going about with method 2:

-   Short Gist for [Deploying a subfolder to GitHub
    Pages](https://gist.github.com/cobyism/4730490)
-   Instructions, tips, tricks and caveats from [the blogdown
    book](https://bookdown.org/yihui/blogdown/github-pages.html)
