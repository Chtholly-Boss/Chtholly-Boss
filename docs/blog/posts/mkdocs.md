---
date:
  created: 2024-11-10
  updated: 2024-11-10
draft: true
---

# MkDocs

This is my first blog post by using **MkDocs**. I will record my learning experience of MkDocs in this post.

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
