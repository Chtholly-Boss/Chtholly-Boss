---
date:
    created: 2024-11-10
    updated: 2024-11-11
categories:
    - Tools
---

# LaTeX
Notes on how to comfortably write $LaTeX$.

<!-- more -->

## Setup
First, we need a comfortable environment to write $LaTeX$ code.

For myself, I installed [MikTeX](https://miktex.org/download) on my Windows 11 machine.
Then, I write $Latex$ in vscode using [Latex Workshop](https://marketplace.visualstudio.com/items?itemName=James-Yu.latex-workshop) extension.

!!! note "Additional Dependencies"
    You may need to install **[Perl](https://strawberryperl.com/)** to make MikTex work with vscode.

After installation, you can create a new `.tex` file and write some code.

```latex
\documentclass{article}
\begin{document}
Hello, world!
\end{document}
```

After you save the file, Latex Workshop will automatically compile it and generate a pdf file.

If you want to use some packages, you can add them in the MikTeX package manager.

## Editing
In Setup, we already write a "Hello World" program. 
However, writing Latex may be annoying when you need to write a lot of code to get something over and over again.
So, we need to learn some tricks to make our life easier.

### Macros
Macros are a powerful feature of Latex. 
You can define a macro to simplify your code.
Macros can be extremely useful when you want to write some formulas over and over again.

```latex
\documentclass{article}
\usepackage{amsmath}
% Define a new command for partial derivatives
\newcommand{\p}[2]{\dfrac{\partial #1}{\partial #2}}
\begin{document}
% using the command
$\p{f}{x}$
\end{document}
```