---
layout: post
title: HOWTO Prettify Coq Code with Listings
tags: latex howto coq
published: True
---

[Listings](http://www.ctan.org/pkg/listings) makes typesetting code easy. The following are some (personal) tips for producing pretty listings for Coq. I'm recording them here mainly as a reminder to my future self.

### Proportional Fonts
First, use a sans serif, proportional font, by setting

``` latex
basicstyle=\sffamily\singlespacing
```

and 

``` latex
columns=fullflexible.
```

For code in C, assembly, etc., I usually use monospaced fonts. But the Coq code I include in my thesis is almost like pseudocode, and proportional fonts are just easier to read.

The \singlespacing ensures that code listings are single spaced, even in the context of a double-spaced document like a thesis. If you're using the memoir class, then you can re-enable the standard setspace package with:

``` latex
\DisemulatePackage{setspace}
\usepackage{setspace}
```

Another option is to use memoir's built-in setspace package (which includes some enhancements, apparently), but I haven't tried this.

### Colored Keywords
Second, color keywords. MidnightBlue is nice.

``` latex
keywordstyle=\bfseries\color{MidnightBlue}
```

I import the color package with these options:

``` latex
\usepackage[usenames,dvipsnames]{color}
```

If you end up using a monospaced font, then you may need

``` latex
\renewcommand{\ttdefault}{pcr}
```

to enable bolding.

### Literate Symbols
Third, use literate symbols (sparingly). 

{% raw %}
<pre>
literate=
  {:=}{{$\defeq\;$}}1
  {->}{{$\rightarrow\;$}}1
  {=>}{{$\Rightarrow\;$}}1
  ...
</pre>
{% endraw %}

For common symbols like ->, =>, etc., this makes code easier to read. But be careful of overruse (which can obscure). A disadvantage is, you can no longer paste code verbatim from the pdf into an editor.

### mathescape
Fourth, mathescape is very handy: 

``` latex
\begin{lstlisting}[mathescape]
$math$
\end{lstlisting}
```

### Centered Listings
Finally, one hack to horizontally center listings is:

``` latex
\begin{center}
\begin{tabular}{c}
\begin{lstlisting}
code
\end{lstlisting}
\end{tabular}
\end{center}
```

There must be a better way, but I haven't found it yet.
