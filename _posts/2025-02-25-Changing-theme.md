---
layout: single
title: Changing Jekyll theme
date: 2025-02-25 12:08:00 +0800
categories: Personal
tags: "Jekyll"
---

Apparently changing the theme of Jekyll is not as easy as I thought.\
Changing the theme and adding gem files along will not build on github page.

>Bundler can't satisfy your Gemfile's dependencies.
>
>Install missing gems with `bundle install'bundle info minimal-mistakes-jekyll.
>
>Warning: The github-pages gem can't satisfy your Gemfile's dependencies. If you want to use a different Jekyll version or need additional dependencies, consider building Jekyll site with GitHub Actions: https://jekyllrb.com/docs/continuous-integration/github-actions/
>
>/usr/local/bundle/gems/jekyll-3.10.0/lib/jekyll/theme.rb:84:in `rescue in gemspec': The minimal-mistakes-jekyll theme could not be found. (Jekyll::Errors::MissingDependencyException)

You need to [use remote theme](https://mmistakes.github.io/minimal-mistakes/docs/quick-start-guide/#remote-theme-method) instead if you want to utilize github pages to build and deploy your site.

