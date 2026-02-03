---
layout: post
title: Moving to Jekyll
categories: blogging jekyll
description: I used to have an old blog on Wordpress. That makes no sense for a static website.
---

As the time went by, I realized that:

 - **Maintenance cost was more than what I expected.** My first deployment was an EC2 with no ALB and a internal database. I am very fond of the "treat you infra as cattle, not pet" principle, so I would need to move away the database to a managed one, while also having a method to deploy the infrastructure easily - ideally, with code;
 - **Wordpress was a liability.** Over the years, Wordpress got a bad rap with the amount of vulnerabilities discovered. A few friends of mine have blogs with were hit because of them;
 - **infrastructure was a liability** Having an EC2 means I would need to take special care to protect it - by updating the OS and libraries, ensuring that SSH is not widely accessible, etc;
 - **I wanted something minimal**. 

I am a big fan of gwern.net, not only by the writing style, but also because the website is aesthetically pleasing. I was surprised that it was actually static content generated from Markdown (https://gwern.net/about#design).

After some research, I found a tool called `jekyll`, which seemed a good option, so I wanted to give it a try.

# Bootstrapping the website using Jekyll

After you install all dependencies (https://jekyllrb.com/docs/installation/ubuntu/), you bootstrap you website with:

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

For this post, the file name is `2025-08-04-moving-to-jekyll.md`

(Reference: https://jekyllrb.com/docs/posts/)

# Viewing your posts

Once a post is ready, you can see how it would look like by executing:

```
jekyll serve
```

# Choosing a theme

The Jekyll community provides multiple themes (https://jekyllrb.com/docs/themes/). My theme of choice was https://github.com/cotes2020/jekyll-theme-chirpy due to its look-and-feel and minimalism.

You can either create a new website using the `jekyll new ` command, then download the content from github and replace it, or start from scratch by cloning the repository above. I choose the latter.

The template only supposed ruby 3.1. I used ruby 3.3.1 for that, managed by rbenv (https://github.com/rbenv/rbenv)

```
rbenv install 3.3.1
export PATH="$HOME/.rbenv/versions/3.3.1/bin:$PATH"
```

# Customizing

You should change the website title and description by changing the corresponding tags on `_config.yml`

# Publishing

I used to have a website on Github Pages, whcih was maintained separately from my Wordpress blog. I now want to merge them together and sue a single repository to publish to both. So now what I do is:
 - Push to kidbomb.github.io
 - Sync changes with `aws s3 sync _site s3://blog.filiperodrigues.me --size-only --storage-class REDUCED_REDUNDANCY`

# Final thoughts
 - Felt like I was on 2005 using Scriptaculous, with no extra bells and whistles
 - Felt like "Arch for blogs"
