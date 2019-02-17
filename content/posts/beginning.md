---
title: "Beginning"
date: 2019-02-13T23:36:55-08:00
draft: false
---

# Notes

I've wanted to write a blog for a while to keep track of my development and
scraps of knowledge. Here's my experience between a few static site
generators I'm deciding on.

* [Hugo](https://gohugo.io)
* [Jekyll](https://jekyllrb.com)
* [Nikola](https://getnikola.com)

## Hugo

I decided to forgoe the prepackaged binary for the fun of using `go get`. I
like to think that I can see all the text scroll by and understand what's
really happening. That new site builder and web server feel super fast!

```bash
$ go get -u -v --tags extended github.com/gohugoio/hugo
$ hugo new site test-site
$ cd test-site
$ hugo server -D  # Same as --buildDrafts
```

O, nothing is served. I see, I need a theme. O, that https://themes.gohugo.io
gallery, talk about eye candy.
[Terminal](https://themes.gohugo.io/hugo-theme-terminal/), I choose you! O,
Git Submodules, I still can't decide if I like you.

```bash
$ git init
$ git submodule add https://github.com/panr/hugo-theme-terminal.git themes/terminal
$ echo 'theme = "terminal"' >> config.toml
$ hugo serve
```

ooOOoo, hey there sexy. Sexy content to go with that sexy theme.

```bash
$ hugo new posts/beginning.md
$ vim content/posts/beginning.md
$ hugo serve -D  # Same as --buildDrafts
```

O! The page automatically refreshes on content changes, what fun! And then to
publish.

```bash
$ hugo
$ ls public/
```

Although, GitHub Pages wants things in `docs/`. Let's avoid the Submodule
example in [Hosting on
GitHub](https://gohugo.io/hosting-and-deployment/hosting-on-github/).

```bash
$ rm -r public/
$ echo 'publishDir = "docs" >> config.toml'
$ hugo
$ git add .
$ git commit -v  # Same as --verbose
$ git push
```

## Jekyll

This seems to be the biggest behemoth at the moment with the backing of
[GitHub Pages](https://pages.github.com). It's a good default, has a large
community, still gets regular releases since 2008 it seems. Ruby ain't so
bad.

Setup was as simple as promised.

```bash
$ gem install bundler jekyll
$ jekyll new test-site
$ cd test-site
$ jekyll serve
```

New post is all about that naming.

```bash
$ vim _posts/2019-02-13-beginning.md
```

Refresh the page and content is there, nice! Publishing seems to be with a
simple `push`.

```bash
$ git add .
$ git commit -v  # Same as --verbose
$ git push
```

## Nikola

Let's use [Pipenv](https://pipenv.readthedocs.io/en/latest/) for the fun of
it.

```bash
$ touch Pipfile
$ pipenv install --skip-lock pip wheel setuptools
$ pipenv install --skip-lock nikola[extras]
$ pipenv run nikola init test-site
# Lots of questions to answer in the wizard
$ mv Pipfile test-site/
$ cd test-site/
$ pipenv run nikola auto
```

Let's create a post.

```bash
$ pipenv run nikola new_post -e
```

O, reStructuredText, I used to use you all the time with Python
documentation. Lately I feel like Markdown. And it looks like by default
`conf.py` comes configured for such a thing, bravo.

```bash
$ vim posts/beginning.md
```

Compiling and `livereload` are a little sluggish, but bearable. The default
theme is basic and, how nice, tries to use the full width of the viewport.

```bash
$ pipenv run nikola build
$ ls output/
```

Hm…not sure how to switch to `docs/`.

# Ending Thoughts

* It seems like everyone wants to show off how fast they with those compile
  times in the server logs

  * Hugo is fastest < Jekyll < Nikola is slowest

* GitHub Pages has some tight integration with Jekyll
