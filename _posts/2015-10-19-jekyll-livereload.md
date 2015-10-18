---
layout: post
title: Combining Jekyll and LiveReload
---

After switching to [github] pages from blogger.com, I really enjoy the ease of using [markdown] to create my blog posts. But there was still room for improvement. Combining [jekyll] and [livereload] now provides almost instant feedback during editing. I found that this is invaluable to me and would like to share the configuration with you.

Jekyll's built-in server already regenerates the site if a source file changes. The other part of my setup is the [guard-livereload] plugin. The [guard] process watches for the changes of a **jekyll** build and will notify [livereload] after the site has been regenerated.

As I am using [bundler] to manage the required dependencies, the `Gemfile` now includes **jekyll** itself and **guard** - plus some utilities to improve the experience on Windows platforms.

```
source "https://rubygems.org"

gem 'jekyll'
gem 'wdm', '>= 0.1.0' if Gem.win_platform?  # native poll for jekyll serve
gem 'guard'
gem 'win32console' if Gem.win_platform?     # colored guard console
gem 'guard-livereload'
```

Guard and its plugins are configured in the `Guardfile`. I instruct the **livereload** plugin to watch the output folder of **jekyll**.

```
guard 'livereload' do
  watch /_site/
end
```

When editing the page both the **jekyll** server and **guard** watcher must run to receive live feedback. I am using a powershell script to conveniently start them in parallel. Note that **jekyll** is configured to render draft posts.

```
Start-Process bundle -ArgumentList "exec jekyll serve --drafts --watch"
Start-Process bundle -ArgumentList "exec guard"
```

Finally I exclude the configuration files from **jekyll** builds in `_config.yml`.

```
exclude: ['Gemfile*', 'Guardfile', '*.ps1' ]
```

You can see the whole setup of my blog in its [github repository](https://github.com/pyranja/pyranja.github.io).

[markdown]: https://daringfireball.net/projects/markdown/
[github]: https://github.com/
[jekyll]: https://jekyllrb.com/
[livereload]: http://livereload.com/
[guard]: https://github.com/guard/guard
[guard-livereload]: https://github.com/guard/guard-livereload
[bundler]: http://bundler.io/
