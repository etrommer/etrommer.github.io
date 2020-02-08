---
layout: post
title: Converting Inkscape SVG for use with LaTeX made easy
categories:
  - Electronics
---

I like using the free vector graphics tool [Inkscape](https://inkscape.org/) to draw diagrams for my thesis. However, embedding them into the LaTeX document can be a bit of a pain. While Inkscape offers a dedicated [LaTeX export option](https://tex.stackexchange.com/questions/151232/exporting-from-inkscape-to-latex-via-tikz) which exports drawing and fonts separately so that the rendering is done in LaTeX, I rarely use this option, as the text typically ends up being not where you expect it, which requires a lot of tuning to make things look right.

You can also export the drawing as a pdf and then export that, however you'll need to do that hand for every change that you make to your drawing, which gets very tedious very fast.

## Shell scripting to the rescue

What I do instead is to make the changes to the Inkscape SVG and have a simple shell script to convert them into a PDF every once in a while using Inkscapes command line interface. 
I then simply have to make my changes to the drawing, save it in Inkscape's native format and the updated version will end up in my document, with no further steps needed.
Here's what my folder structure looks like:

```
project
    ├── img
    │   ├── convert_svg.sh
    │   ├── pdf
    │   │   └── some_image.pdf
    │   └── svg
    │       └── some_image.svg
    └── project.tex
```

and here's the content of the script:

```bash
#!/bin/bash
for file in svg/*.svg;
do
    echo "Converting: $file"
    output_name="pdf/$(basename -s .svg $file).pdf"
    inkscape --export-pdf=$output_name --export-area-drawing $file
done
```
It simply converts all the file under `svg` into a PDF of the same name in the folder `pdf`. 

If I add a new file, I'll have to run the script manually once. I get my drawing converted into a PDF which I can then easily embed without issues.
```latex
\begin{figure}[htb!]
	\includegraphics[width=\linewidth]{img/pdf/some_drawing.pdf}
	\centering
	\caption{Some drawing}
\end{figure}
```
I have the script attached to a git pre-commit hook, so that all my files get updated automatically when I commit a change. Could certainly also be done with a Makefile if the process ever ends up taking a more significant amount of time than it is now.
