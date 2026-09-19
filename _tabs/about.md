---
# the default layout is 'page'
icon: fas fa-info-circle
order: 3
---

I'm Filipe Paz Rodrigues. I write about security, smartcards, and embedded
devices, and I use this blog to work through things in more depth than a
thread or a set of notes allows.

Most of what ends up here starts the same way: I read about something that
sparks my interest, then I start writing about it. I make an effort for
all of my blog posts to be independently verifiable by providing source code
and references.

> **Disclaimer:** I use large language models as part of my writing process.
> I am the one that always starts the draft, while AI helps me with additional
> information, restructuring, and catching mistakes. The code,
> the commands, their output, and the sources are always independently
> verified before a post is published.
{: .prompt-info }

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

## Open source

Most of my open source work has been on the same problem from different
angles: making USB smartcard readers work on embedded hardware. The
[CCID](https://en.wikipedia.org/wiki/CCID_(protocol)) specification is how a
host talks to a smartcard reader over USB, and it tends to be the missing
piece in otherwise capable USB stacks.

- **[Flipper Zero](https://github.com/flipperdevices/flipperzero-firmware)** —
  I added CCID support, starting with [initial USB CCID
  support](https://github.com/flipperdevices/flipperzero-firmware/pull/3048)
  in 2023 and building it up over about a dozen pull requests: [ISO7816
  support](https://github.com/flipperdevices/flipperzero-firmware/pull/3681),
  a [worker
  thread](https://github.com/flipperdevices/flipperzero-firmware/pull/3785),
  and [interrupt
  support](https://github.com/flipperdevices/flipperzero-firmware/pull/4198).
  It now ships as a standalone app in the [official application
  catalog](https://github.com/flipperdevices/flipper-application-catalog/pull/1163),
  with the source at
  [kidbomb/flipper-ccid-app](https://github.com/kidbomb/flipper-ccid-app).
- **[LUFA](https://github.com/abcminiuser/lufa)** — the CCID class driver for
  this AVR USB library, contributed in 2018: [low-level and class driver
  changes](https://github.com/abcminiuser/lufa/pull/126), [PC-to-Reader
  XfrBlock](https://github.com/abcminiuser/lufa/pull/131), and [Get/Set
  Parameters](https://github.com/abcminiuser/lufa/pull/135).
- **[libusb_stm32](https://github.com/flipperdevices/libusb_stm32)** — [CCID
  support](https://github.com/flipperdevices/libusb_stm32/pull/1) for the STM32
  USB device stack.
- **[Trivy Operator](https://github.com/aquasecurity/trivy-operator)** — a
  [fix for client/server mode
  scans](https://github.com/aquasecurity/trivy-operator/pull/1344) in Aqua
  Security's Kubernetes scanner.

Alongside those, a few smaller things of my own: a [smartcard simulator on an
ATtiny85](https://github.com/kidbomb/sc-attiny), a [smartcard built as a
PCB](https://github.com/kidbomb/sc-pcb), and
[tooling](https://github.com/kidbomb/neander) for the Neander educational
processor.

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
