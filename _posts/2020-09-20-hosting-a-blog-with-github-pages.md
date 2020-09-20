---
layout: post
title:  "Hosting a blog with Github Pages"
permalink: /2020/09/20/hosting-a-blog-with-github-pages
tags: jekyll github
---

Hosting a blog on Github Pages is extremely simple and completely free. We will 
walk through the creation process of this blog in this post.

## Create a Github repository

To use Github pages, we need a Gitbub repository, so let's go ahead and create
an empty repository called `blog`. After that, follow the instructions provided
to clone the repository onto our computer and create a `gh-pages` branch as the
source of our site content.

{% highlight bash %}
git checkout --orphan gh-pages
{% endhighlight %}

## Create a new Jekyll site

Jekyll is a blog-aware static site generator that is supported directly by
Github pages, so we will be using it here. To avoid messing with all the Ruby
dependencies of Jekyll, we will be using Jekyll through Docker. Run the
following Docker command to create a new Jekyll site in the current directory.
The official Jekyll image will be pulled down, and the Jekyll files will be
created right on our host machine thanks to Docker's [bind mount][bind-mount].

{% highlight bash %}
docker run \
  --rm \
  -it \
  --volume="$PWD:/srv/jekyll" \
  jekyll/jekyll:3.8 jekyll new .
{% endhighlight %}

In `Gemfile`, uncomment and change the following line:

{% highlight diff %}
- # gem "github-pages", group: :jekyll_plugins
+ gem "github-pages", "~> 207", group: :jekyll_plugins
{% endhighlight %}

Also, remove `Gemfile.lock` so that the dependencies can be updated
automatically.

## Test the site locally

Again, we can use Docker to run `jekyll serve` for testing the site locally.
`$PWD/vendor/bundle` is used to cache the gems so that we don't have to fetch
them again and again. Make sure this directory is ignored by git and Jekyll.

{% highlight bash %}
docker run \
  --rm \
  --volume="$PWD:/srv/jekyll" \
  --volume="$PWD/vendor/bundle:/usr/local/bundle" \
  -p 4000:4000 \
  -it \
  jekyll/jekyll:3.8 jekyll serve
{% endhighlight %}

Navigating to http://localhost:4000/ and we should be able to see the blog up
and running.

## Deploy to Github Pages

Deploying to Github Pages is as simple as pushing all the files to the
repository. Github will manage building the Jekyll site automatically.
To confirm that it is working correctly, go to Settings of the Github
repositories. Under Github Pages, Branch: gh-pages should be automatically
selected as the Source, and it should show that "Your site is ready to be
published at https://<username>.github.io/blog." 

## Choose a theme

While we are at it, let's also choose a theme from the numerous freely provided
ones on Github. Click on "Change Theme" under Github Pages settings and apply
your favorite theme to the blog.

*Note: If the blog becomes empty after applying a theme, please check whether
the layout names have been changed in the selected theme and update the existing
pages to use the new layout names accordingly*

## Start blogging

Without much work, we already have a nice looking blog hosted for free, so that
we can dive right into writing our blog posts. Each blog post is just a markdown
file inside the `_posts` directory. With `jekyll serve` still running, We can just
edit the files and immediately see the results in the browser. Once we are done
editing, just push all the files again to Github and the updates will be available
quickly.

[bind-mount]: https://docs.docker.com/storage/bind-mounts/
