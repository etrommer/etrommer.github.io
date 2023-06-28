+++
title = "Exporting InkScape drawings with zero clicks"
+++
Workflow for automatically converting InkScape SVGs into PDFs that can seamlessly be included in LaTeX drawings.
<!-- more -->

I like using [Inkscape](https://inkscape.org/) to draw diagrams for my thesis. However, embedding them into the LaTeX document can be a bit of a pain. While Inkscape offers a dedicated [LaTeX export option](https://tex.stackexchange.com/questions/151232/exporting-from-inkscape-to-latex-via-tikz) which exports drawing and fonts separately so that the rendering is done in LaTeX, I rarely use this option, as the text typically ends up being not where you expect it, which requires a lot of tuning to make things look right.

You can also export the drawing as a pdf manually and then use that in your LaTeX document, however you'll need to do that by hand every time you change something. This gets very tedious, very fast if you are like me and tend to over-optimize your drawings through countless iterations.

## Solution 1: Shell scripting to the rescue

The first solution that I've come up with instead is to make the changes to the Inkscape SVG and have a separate shell script to convert them into a PDF every once in a while using Inkscape's command line interface. 
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
Drawings go in the `svg` folder.
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

## Solution 2: Makefile

One downside to this is that each file is generated once the script is run, regardless of whether it was updated or not. While this is not a major deal because converting only takes a few seconds, there is an issue when using the files in Git. Git will pick up all of the PDFs and list them as unstaged changes, even though nothing has changed with them. The solution to that is to only convert the files when there are actual changes. The perfect job for a Makefile!
Here's the content of the Makefile that I have in the `img` folder:
```make
# Inkscape SVG to PDF Makefile

# Input directory
SVGDIR = svg

# Output directory
PDFDIR = pdf

# Collect SVGs from input folder
SVGS = $(wildcard $(SVGDIR)/*.svg)
PDFS = $(patsubst $(SVGDIR)/%.svg, $(PDFDIR)/%.pdf, $(SVGS))

# Build recipe for PDFs
$(PDFDIR)/%.pdf: $(SVGDIR)/%.svg
        inkscape --export-pdf=$@ --export-area-drawing $^

# Create output dir if necessary
$(PDFDIR):
        mkdir -p $@

.PHONY: all clean

all: $(PDFDIR) $(PDFS)

clean:
        @echo "Removing previously generated PDFs"
        rm -rf $(PDFDIR)
        @echo "Done."
```
From the top-level directory, I can then convert all my updated SVGs at once with `make -C img all`.
## Workflow

Whenever I add a new file, I'll have to run the makefile manually once. I get my drawing converted into a PDF which I can then easily embed like so:
```latex
\begin{figure}[htb!]
	\includegraphics[width=\linewidth]{img/pdf/some_drawing.pdf}
	\centering
	\caption{Some drawing}
\end{figure}
```
I also have the script attached to git pre-commit hook that converts all SVGs automatically before every commit. That way, my SVG changes will always be reflected in my document without any additional trouble caused by converting files back an forth. I'm generally very happy with this solution!
