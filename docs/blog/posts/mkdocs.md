---
date:
  created: 2024-11-10
  updated: 2024-11-13
categories:
    - Tools
---

# MkDocs

Notes on learning **MkDocs**.

<!-- more -->

To install MkDocs, you can use pip:
```bash
pip install mkdocs
```

## Hello, MkDocs
When I first access a tool, I usually want to know how to use it to do sth simple but enough to construct a naive conceptual model. 
For MkDocs, I follow [mkdocs-material](https://squidfunk.github.io/mkdocs-material/getting-started/) a great tutorial to get started.

### Init 
To create a new MkDocs project, you can use the following command:
```bash
mkdocs new [directory]
```

This will create a new directory with the specified name and a `mkdocs.yml` configuration file. 
Most configurations will go in the `mkdocs.yml` file. For now, we just keep it simple and add a site_name:
```yaml
site_name: My Docs
```

Now, we must get something to see what will happen when we modify anything. This can be done by:
```bash
mkdocs serve
```

This will start a local web server and open the default web browser to the MkDocs site. 
Go to `http://127.0.0.1:8000/` to see the site.

Before building the site, we may want to know how to write blogs and how to organize them.

### Writing
I refer to [Blog Tutorial of mkdocs-material](https://squidfunk.github.io/mkdocs-material/tutorials/blogs/basic/) to learn how to write blogs.

First, modify the `mkdocs.yml` file like the following:
```yaml
site_name: ...
theme:
  name: material
plugins:
  - search
  - blog
```
Then you will see a new `docs/blog` directory in your project. Blogs and relevant files are stored in this directory.
Each post must have a page header with at least date fields. For example:

```md
---
date: 
  created: 2024-11-10
---

# Hello, MkDocs
This is the excerpt that will show up in the blog list.
<!-- more -->
Follows will not be shown in the excerpt.
```

You can set many metadata for each post inside the header:

- draft: if set to true, `build` will ignore it
- readtime: the estimated reading time of the post
- pin: if set to true, the post will be pinned to the top of the index page
- links: a list of links to other pages or resources related to the post

### Deploying
Refer to [Deploying to GitHub Pages](https://www.mkdocs.org/user-guide/deploying-your-docs/#project-pages) to learn how to deploy your site using Github Pages.

## Misc Topics
### Navigation
When we write a lot of blogs, we may want to organize them in a tree structure.
To do this, we can use the `nav` field in the `mkdocs.yml` file. For example:
```yaml
nav:
  - Home: index.md
  - Blog:
    - blog/index.md
    ...
```
Navigation can be extremely useful when combined with the *Categories* and *Tags* feature of MkDocs. See [Navigation tutorial of mkdocs-material](https://squidfunk.github.io/mkdocs-material/tutorials/blogs/navigation/#using-categories) for more details.

To put it easy, what you need is just to add a `categories:` field in the header of the post.
Also, in the `mkdocs.yml` file, you can write:
```yaml
plugin:
  ...
  blog:
    ...
    categories_name: Categories
    categories_allowed:
        - Tools
        ...
```

And besides categories, you may also want to use tags to classify your posts.
First, add a tags plugin in `mkdocs.yml`:
```yaml
plugins:
  - blog:
    ...
  - tags
```
Then add a `tags:` field in the header of the post:
```md
---
date: ...
tags:
  - Python
---
```
Finally, add a tag index file:
- create a `docs/blog/tags.md` file
- modify `mkdocs.yml`:

```yaml
plugins:
  - tags:
    tags_file: blog/tags.md
nav:
  - Blog:
    - blog/index.md
    - Tags: blog/tags.md
```


### Math
From time to time, I will need to write math equations in my blog. Markdown itself support LaTeX locally, but MkDocs does not support it by default.
So, I refer to [mkdocs-material Math Reference](https://squidfunk.github.io/mkdocs-material/reference/math/) to learn how to accomplish this.

To put it easy, first, inside `docs/`, create a `javascripts/` directory and put a `mathjax.js` file inside it.
Fill the file with the following content:
```js
window.MathJax = {
  tex: {
    inlineMath: [["\\(", "\\)"]],
    displayMath: [["\\[", "\\]"]],
    processEscapes: true,
    processEnvironments: true
  },
  options: {
    ignoreHtmlClass: ".*|",
    processHtmlClass: "arithmatex"
  }
};

document$.subscribe(() => { 
  MathJax.startup.output.clearCache()
  MathJax.typesetClear()
  MathJax.texReset()
  MathJax.typesetPromise()
})
```
Then, inside `mkdocs.yml`, add the following lines:
```yaml
...
markdown_extensions:
  - pymdownx.arithmatex:
      generic: true

extra_javascript:
  - javascripts/mathjax.js
  - https://unpkg.com/mathjax@3/es5/tex-mml-chtml.js
```

### Admonitions
Admonitions are a way to highlight important information in your documentation.
You can write an admonition like the following:
```md
!!! [type] "Title"
    Content
```

I often use the following admonitions:

!!! tip "Tip"
    This is a tip.

!!! note "Note"
    This is a note.

!!! warning "Warning"
    This is a warning.
