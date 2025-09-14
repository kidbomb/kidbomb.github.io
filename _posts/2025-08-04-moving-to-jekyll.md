---
layout: post
title: Moving to Jekyll
categories: blogging jekyll
tags: blogging jekyll wordpress github-pages
description: Why I have moved from hosting a blog on WordPress to static hosting
---

# Introduction

I like to write my thoughts and research from time to time, and I also like to share them with the world. I also prefer simple and easy-to-maintain systems.

My old blog stack used an EC2 instance with WordPress. This might look simple at first glance, but it requires ongoing maintenance. You need to keep your EC2 instance updated with the latest patches, and the same applies to your WordPress deployment. On top of that, there is the baseline compute cost for resources that remain underutilized most of the time. The attack surface is also larger: you need to handle security controls for EC2 and the backend database correctly.

Some well-known bloggers, such as Gwern from gwern.net, use static content generated from Markdown. That addresses my concerns around security and simplicity: with this setup, I can host my blog using a single S3 bucket. For additional security and performance, I can also use CloudFront.

Jekyll was the no-brainer solution. It is widely used and also supported by GitHub Pages. Since I already had a page there, this gave me a way to converge everything.

# Bootstrapping the website using Jekyll

After you install all dependencies (https://jekyllrb.com/docs/installation/ubuntu/), you can bootstrap your website with:

```
jekyll new site/ --blank
```

That creates a folder with the following structure:

```
.
├── _config.yml
├── _data
│   └── members.yml
├── _drafts
├── _includes
│   ├── footer.html
│   └── header.html
├── _layouts
│   ├── default.html
│   └── post.html
├── _posts
├── _sass
│   ├── _base.scss
│   └── _layout.scss
├── _site
├── .jekyll-cache
│   └── Jekyll
│       └── Cache
│           └── [...]
├── .jekyll-metadata
└── index.html # can also be an 'index.md' with valid front matter
```
(Reference: https://jekyllrb.com/docs/structure/)

# Creating a post

The `_posts` folder is where all your posts go. To create one, add a file with the following format:

```
YEAR-MONTH-DAY-title.MARKUP
```

For this post, the file name is `2025-08-04-moving-to-jekyll.md`.

(Reference: https://jekyllrb.com/docs/posts/)

# Viewing your posts

Once a post is ready, you can see how it looks by executing:

```
jekyll serve
```

# Choosing a theme

The Jekyll community provides multiple themes (https://jekyllrb.com/docs/themes/). My theme of choice was https://github.com/cotes2020/jekyll-theme-chirpy due to its look-and-feel and minimalism.

You can either create a new website using the `jekyll new` command and then replace its content, or start from scratch by cloning the repository above. I chose the latter.

The template officially supports Ruby 3.1. I used Ruby 3.3.1, managed with rbenv (https://github.com/rbenv/rbenv).

```
rbenv install 3.3.1
export PATH="$HOME/.rbenv/versions/3.3.1/bin:$PATH"
```

# Customizing

You should change the website title and description by editing the corresponding fields in `_config.yml`.

# Publishing

I used to have a website on GitHub Pages, which was maintained separately from my WordPress blog. I wanted to merge them and use a single repository to publish to both. So now I:

 - Push to kidbomb.github.io
 - Sync changes with `aws s3 sync _site s3://blog.filiperodrigues.me --size-only --storage-class STANDARD_IA`

# Final thoughts

Removing EC2 and WordPress from my blog stack was a relief. Using Jekyll saves me a lot of time and helps me focus on writing and design. Customizations still require extra effort, which sometimes feels like "Arch Linux for blogs" or reviving older web stacks—but that tradeoff is intentional. Jekyll stays minimal while still being extensible, which is exactly what I need right now.
