---
# the default layout is 'page'
icon: fas fa-info-circle
order: 3
---

I'm Filipe Paz Rodrigues. I write about security, smartcards, and embedded
devices, and I use this blog to work through things in more depth than a
thread or a set of notes allows.

Most of what ends up here starts the same way: I read about something that 
sparkles my interest, then I start writing about it. I make an effort for 
all of my blog posts to be independently verifyable by providing source code
and references.

## What's here

Two series so far, both best read in order:

- [**Container runtimes**](/tags/container-runtimes/) — what container
  runtimes actually do, starting from OCI and CRI fundamentals and working
  down through `runc`'s lifecycle: namespaces, the init process, the rootfs,
  capabilities, seccomp, and the OCI hooks.
- [**CVE-2025-23266**](/tags/nvidia/) — a two-part look at the NVIDIA
  Container Toolkit escape, including how to reproduce it and why
  `createContainer` hooks made it possible.

Everything is indexed under [tags](/tags/) and [archives](/archives/).

## Elsewhere

- GitHub: [kidbomb](https://github.com/kidbomb) and
  [filipepazrodrigues](https://github.com/filipepazrodrigues)
- X/Twitter: [@kidbomb](https://twitter.com/kidbomb)
- Keybase: [kidbomb](https://keybase.io/kidbomb)

For anything sensitive, my PGP key is on Keybase and the proof is at
[/keybase.txt](/keybase.txt):

```
A0E3 5546 1DBD 974A 9D57  BFF6 09B5 836B F3FA F8B8
```

## About the site

Static, built with [Jekyll](https://jekyllrb.com/) and the
[Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) theme, published
to GitHub Pages and mirrored to S3. I wrote up the reasoning, and the move
off WordPress that prompted it, in [Moving to
Jekyll](/posts/moving-to-jekyll/).
