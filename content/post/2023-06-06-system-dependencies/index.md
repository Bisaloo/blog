---
slug: system-dependency
title: "System Dependencies in R Packages & Automatic Testing" 
authors: 
- Hugo Gruson
date: "2023-06-06" 
tags: 
- package development 
- r-package
output: hugodown::hugo_document
rmd_hash: 0fc3e5194402088f

---

<!-- FIXME: Use this image for social media: https://pixabay.com/fr/photos/alice-anglais-pays-des-merveilles-2902560/ -->

In a previous post, we discussed about a package dependency that goes slightly beyond the normal R package ecosystem dependency: R itself. Today, we step even further and discuss about dependencies that have nothing to do with R: system dependencies. In particular, we are going to talk about system dependencies in the context of automated testing: is there anything extra to do when setting continuous integration for your package with system dependencies? How does it work behind the scenes? And how to work with edge cases?

## Introduction: specifying system dependencies in R packages

Before jumping right into the topic of continuous integration, let's take a moment to introduce, or remind you, how system dependencies are specified in R packages. You can directly jump to the next session if this concept is fresh in your memory.

Let's imagine we have a package depending on The official 'Writing R Extensions' guide states [^1]:

> Dependencies external to the R system should be listed in the 'SystemRequirements' field, possibly amplified in a separate README file.

One important thing to note is that this field contains free text. As such, to refer to the same piece of software, you could write either one of the following in the package `DESCRIPTION`:

``` yaml
SystemRequirements: ExternalSoftware
```

``` yaml
SystemRequirements: ExternalSoftware 0.1
```

``` yaml
SystemRequirements: lib-externalsoftware
```

## The general case: everything works automagically

If while reading the previous section, you could already sense the problems linked to the fact `SystemRequirements` is a free-text field, fret not! In the very large majority of cases, setting up continuous integration in an R package with system dependencies is exactly the same as with any other R package.

Using, as often, the supercharged usethis package, you can automatically create the relevant GitHub Actions workflow file in your project [^2]:

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'><span><span class='nf'>usethis</span><span class='nf'>::</span><span class='nf'><a href='https://usethis.r-lib.org/reference/use_github_action.html'>use_github_action</a></span><span class='o'>(</span><span class='s'>"check-standard"</span><span class='o'>)</span></span></code></pre>

</div>

The result is:

``` yaml
# Workflow derived from https://github.com/r-lib/actions/tree/v2/examples
# Need help debugging build failures? Start at https://github.com/r-lib/actions#where-to-find-help
on:
  push:
    branches: [main, master]
  pull_request:
    branches: [main, master]

name: R-CMD-check

jobs:
  R-CMD-check:
    runs-on: ${{ matrix.config.os }}

    name: ${{ matrix.config.os }} (${{ matrix.config.r }})

    strategy:
      fail-fast: false
      matrix:
        config:
          - {os: macos-latest,   r: 'release'}
          - {os: windows-latest, r: 'release'}
          - {os: ubuntu-latest,   r: 'devel', http-user-agent: 'release'}
          - {os: ubuntu-latest,   r: 'release'}
          - {os: ubuntu-latest,   r: 'oldrel-1'}

    env:
      GITHUB_PAT: ${{ secrets.GITHUB_TOKEN }}
      R_KEEP_PKG_SOURCE: yes

    steps:
      - uses: actions/checkout@v3

      - uses: r-lib/actions/setup-pandoc@v2

      - uses: r-lib/actions/setup-r@v2
        with:
          r-version: ${{ matrix.config.r }}
          http-user-agent: ${{ matrix.config.http-user-agent }}
          use-public-rspm: true

      - uses: r-lib/actions/setup-r-dependencies@v2
        with:
          extra-packages: any::rcmdcheck
          needs: check

      - uses: r-lib/actions/check-r-package@v2
        with:
          upload-snapshots: true
```

You may notice there is no explicit mention of system dependencies in this file. Yet, if we use this workflow in an R package with system dependencies, everything will work out-of-the-box in most cases. So, when are system dependencies installed? And how the workflow does even know which dependencies to install since the `SystemRequirements` is free text that may not correspond to the exact name of a library?

The magic happens in the `r-lib/actions/setup-r-dependencies` step. If you want to learn about it, you can read the source code of this step. It is mostly written in R but it contains a lot of bells and whistles to handle messaging within the GitHub Actions context and as such, it would be too long to go through it line by line in this post. However, at a glance, you can notice many mentions of the [pak R package](https://pak.r-lib.org/).

If it's the first you're hearing about the pak package, we strongly recommend we go through the [list of the most important pak features](https://pak.r-lib.org/reference/features.html). It is packed with many very powerful feature. The specific feature we're interested in here is the automatic install of system dependencies via [`pak::pkg_sysreqs()`](https://pak.r-lib.org/reference/local_system_requirements.html), which in turn uses `pkgdepends::sysreqs_install_plan()`.

We now understand more precisely where the magic happens but it still doesn't explain how pak is able to know which precise piece of software to install from the free text `SystemRequirements` field. As often when you want to increase your understanding, it is helpful to [read the source](https://blog.r-hub.io/2019/05/14/read-the-source/). While browsing pkgdepends source code, we see a call to <https://github.com/r-hub/r-system-requirements>.

This repository contains a set of [rules](https://github.com/rstudio/r-system-requirements/tree/main/rules) as json files which match unformatted software name via regular expressions to the exact libraries for each major operating system. Let's walk through an example together:

``` json
{
  "patterns": ["\\bnvcc\\b", "\\bcuda\\b"],
  "dependencies": [
    {
      "packages": ["nvidia-cuda-dev"],
      "constraints": [
        {
          "os": "linux",
          "distribution": "ubuntu"
        }
      ]
    }
  ]
}
```

The regular expression tells that each time a package lists something as `SystemRequirements` with the word "nvcc" or "cuda", the corresponding Ubuntu library to install is `nvidia-cuda-dev`.

## When it's not working out-of-the-box

We are now realizing that this automagical setup we didn't pay so much attention to until now actually requires a very heavy machinery under the hood. And it happens, very rarely, that this complex machinery is not able to handle your specific use case. But it doesn't mean that you cannot use continuous integration in your package. It means that some extra steps might be required to do so. Let's review these possible solutions together in order of complexity.

### Fix it for everybody by submitting a pull request

One first option might be that the regular expression used by `r-hub/r-system-requirements` to convert the free text in `SystemRequirements` to a library distributed by your operating system does not recognize what is in `SystemRequirements`.

To identify if this is the case, you need to find the file containing the specific rule for the system dependency of interest in `r-hub/r-system-requirements`, and test the regular expression on the contents of `SystemRequirements`.

If we re-use the cuda example from the previous section and we are wondering why it is not automatically installed for a package specifying "cudaa":

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'><span><span class='nf'>stringr</span><span class='nf'>::</span><span class='nf'><a href='https://stringr.tidyverse.org/reference/str_match.html'>str_match</a></span><span class='o'>(</span><span class='s'>"cudaa"</span>, <span class='nf'><a href='https://rdrr.io/r/base/c.html'>c</a></span><span class='o'>(</span><span class='s'>"\\bnvcc\\b"</span>, <span class='s'>"\\bcuda\\b"</span><span class='o'>)</span><span class='o'>)</span></span>
     [,1]
[1,] NA  
[2,] NA  
</code></pre>

</div>

This test confirms that the `SystemRequirements` field contents are not recognized by the regular expression. Depending on the case, the best course of action might be to:

-   either edit the contents of `SystemRequirements` so that it's picked up by the regular expression
-   or submit a pull request to `r-hub/r-system-requirements` if you believe the regular expression is too restrictive and should be updated

Note however that the first option is likely always the simplest as it doesn't impact all the rest of the ecosystem (which is why `r-system-requirements` maintainers might be reluctant to relax a regular expression) and it is often something directly in your control, rather than a third-party.

### Install system dependencies manually

However, you might be in a case where the system dependency to install is not provided by package managers at all. Typically, if you had to compile or install it manually on your local computer, you're very likely to have to do the same operation in GitHub Actions. There two different, but somewhat equivalent, ways to do so, as detailed below.

#### Directly in the GitHub Actions workflow

You can insert the installation steps you used locally in the GitHub Actions workflow file. So, instead of having the usual structure, you have an extra step "Install extra system dependencies manually" that may look something like this:

``` diff
jobs:
  R-CMD-check:
    runs-on: ubuntu-latest
    env:
      GITHUB_PAT: ${{ secrets.GITHUB_TOKEN }}
      R_KEEP_PKG_SOURCE: yes
    steps:
      - uses: actions/checkout@v3

      - uses: r-lib/actions/setup-r@v2
        with:
          use-public-rspm: true

+      - name: Install extra system dependencies manually
+        run:
+          wget ...
+          make
+          sudo make install

      - uses: r-lib/actions/setup-r-dependencies@v2
        with:
          extra-packages: any::rcmdcheck
          needs: check

      - uses: r-lib/actions/check-r-package@v2
```

#### Using a Docker image in GitHub Actions

Alternatively, you can do the manual installation in a Docker image and use this image in your GitHub Actions workflow. This is a particularly good solution if there is already a public Docker image or you already wrote a `DOCKERFILE` for your own local development purposes. If you use a public image, you can follow the steps in the official documentation to integrate it to your workflow. If you use a `DOCKERFILE`, you can follow [the answers to this stackoverflow question](https://stackoverflow.com/q/61154750/4439357) (in a nutshell, use `docker compose` in your job or publish the image first and then follow the official documentation).

## Conclusion

In this blog, we have quickly reviewed how to specify system requirements for R package, how this seemingly innocent task requires a very complex infrastructure so that it can be understood by automated tools and that your dependencies are smoothly installed in a single command. We also gave some pointers on what to do if you're in one of the rare cases where the automated tools don't or can't work.

One final note on this topic is that there might be a move from CRAN to start requiring more standardization in the `SystemRequirements` field. One R package developer has reported being asked to change "Java (\>= 8)" to "Java JRE 8 or higher". In all cases, it is probably good practice to do this on your own and check what other R packages with similar system dependencies are writing in `SystemRequirements`.

[^1]: For R history fans, this has been the case [since R 1.7.0](https://github.com/r-devel/r-svn/blob/9c46956fd784c6985867aca069b926d774602928/doc/NEWS.1#L2348-L2350).

[^2]: Alternatively, if you're not using usethis, you can manually copy-paste the relevant GitHub Actions workflow file from the `examples` of the `r-lib/actions` project.
