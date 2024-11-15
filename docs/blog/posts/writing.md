---
date:
    created: 2024-11-10
    updated: 2024-11-15
categories:
    - Tools
tags:
    - Writing
pin: true
---

# Writing Tools
Here are some tools that I use for writing.

<!-- more -->

## Why different tools?
When I mentioned writing, I am talking about:
- Writing daily notes
- Writing a post/document
- Writing a paper
- Writing a slide

For different purposes, I need different tools. Just as the UNIX philosophy:
!!! quote "Do One Thing and Do It Well"

During the experience of using one tool, you may find that it is suitable for one part of your work, but not for another part.
So, you need to use different tools for different purposes.

## Environment
Most of the time, I use [Visual Studio Code](https://code.visualstudio.com/) as my **Writing IDE**, mainly because its **Extensibility**.
That is, you can install extensions to make it do whatever you want and enhance your experience.

### Markdown
I use [Markdown](https://www.markdownguide.org/) to write daily notes and posts.
You can quickly setup a local Markdown environment by installing the following extensions:
- [Markdown All in One](https://marketplace.visualstudio.com/items?itemName=yzhang.markdown-all-in-one)
- [Markdown Preview Enhanced](https://marketplace.visualstudio.com/items?itemName=shd101wyy.markdown-preview-enhanced)

You could quickly master Markdown by reading the [official guide](https://www.markdownguide.org/basic-syntax/). For writing posts, 
you can refer to my other posts like:
- [mkdocs](../posts/mkdocs.md)

### Latex
I sometimes use [Latex](https://www.latex-project.org/) to write documents and papers.

You can quickly setup a local Latex environment by [Latex Workshop](https://marketplace.visualstudio.com/items?itemName=James-Yu.latex-workshop) extension.

For this extension to work, you need to:
- Install a Latex distribution, I choose [MikTex](https://miktex.org/download)
- If **MikTex** is installed, you need to install **[Perl](https://strawberryperl.com/)** to make it work with vscode.

After the insallation, you can create a new `.tex` file and write some code.

```latex
\documentclass{article}
\begin{document}
Hello, world!
\end{document}
```
After you save the file, Latex Workshop will automatically compile it and generate a pdf file.
If you want to use some packages, you can add them in the MikTeX package manager.

### Typst
I use [Typst](https://typst.dev/) for writing my most homework assignments, and sometimes for writing some slides.

You can quickly setup a local Typst environment by [Tinymist Typst](https://marketplace.visualstudio.com/items?itemName=myriad-dreamin.tinymist) extension.

After the insallation, you can create a new `.typ` file and write some code.

```typ
= Hello, world!
```

I have written a post about Typst [here](../posts/typst.md).
