---
title: "Choosing Hugo and GitLab Pages"
date: 2021-08-02T11:30:00-07:00
---

It seems reasonable, in the spirit of self-documentation, for the first
real post to this site to describe the creation of the site itself.

### Hosting

The precipitating event for creating this site was my finding out about
the [.ooo TLD](https://icannwiki.org/.ooo), which formed a clear [domain
hack](https://en.wikipedia.org/wiki/Domain_hack) with my family name.

I used [Google Domains](https://domains.google) to test URLs for
availability and eventually settled on [this one](https://etw.ooo),
then completed the purchase through Google Domains.

I chose [GitLab Pages](https://docs.gitlab.com/ee/user/project/pages)
for hosting. I considered [GitHub Pages](https://pages.github.com)
initially, but when I could neither access my GitHub account (second
factor and recovery codes all failed) nor recover it (still waiting on
response from GitHub support), I went with a different option.

### Site Generation

I tested a few static site generators and their online HOWTO guides.

I first tried [Zola](https://www.getzola.org). After completing the
[HOWTO](https://www.getzola.org/documentation/getting-started/overview),
I could not get themes to work. I probably messed up my `config.toml`.

I then tried [Jekyll](https://jekyllrb.com). After completing the
[HOWTO](https://jekyllrb.com/docs), I could not get themes to work. I
likely used `bundler` incorrectly. After fussing with my `Gemfile`, it
seemed unlikely that the resulting local configuration was reproducible
and further doubtful that a CI/CD pipeline would emit matching results.

At this point, I took a break, planning to try one more system before
falling back to [writing raw HTML](https://motherfuckingwebsite.com).

A day or two later, I tried [Hugo](https://gohugo.io/). After completing
the [HOWTO](https://gohugo.io/getting-started/quick-start), I was able
to get [themes](https://github.com/vaga/hugo-theme-m10c) to work. [Integrating
with GitLab Pages](https://gitlab.com/ewoo/www/-/blob/main/.gitlab-ci.yml)
worked [as doc-ed](https://gohugo.io/hosting-and-deployment/hosting-on-gitlab).

Configuring the [custom domain](https://docs.gitlab.com/ee/user/project/pages/custom_domains_ssl_tls_certification)
required a few attempts in the GitLab UI and Google Domains UI, either
because of user error, DNS propagation times, or both.
