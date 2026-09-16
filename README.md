# Tales of OpenEverest

Source for [talesofopeneverest](https://hamidkhan1001.github.io/TalesOfOpenEverest/), a running log of learning Go and open source by contributing to [OpenEverest](https://github.com/openeverest/openeverest).

Built with Jekyll, hosted on GitHub Pages. No CMS, no automation pulling from the GitHub API: every post is a real file, written after the PR it describes actually happened.

## Running locally

```
bundle install
bundle exec jekyll serve
```

## Structure

- `_posts/`: one post per PR/issue I've actually worked, in order
- `architecture/`: my own working map of the OpenEverest codebase, corrected as I understand more
- `learning-go/`: Go concepts, added only once I've hit them in real code
