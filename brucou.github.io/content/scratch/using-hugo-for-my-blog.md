---
title: "Using Hugo for my blog"
date: 2017-05-09T08:06:49+01:00
# lastmod: {{ .Date }}
draft: false
tags: ["hugo"]
categories: ["lessons-learnt"]
author: "brucou"
toc: false
---

# Background
Hugo is a static website generator, in the same vein as Jekyll or Hexo. A website can be understood as :

- a server with the usual concerns (routing, authentication, content delivery, etc.)
- hosted content (html, js, css, etc.)

A static website generator has, as its mission, the generation of the hosted content from other content. Hence it can be specified by `f`, a function so that, given a content `c`, returns the hosted content constituting the target website : `f(c) = h`. 

Our website is hence a set of files capturing different concerns : $website 
=\bigcup_{concerns}files$. For instance, $website = server \bigcup javascript \bigcup css \bigcup
 html$. Each of those concerns can be subject to a generator. For instance, `css` files can be 
 generated from `scss` files. We will focus here on `html` files generation, which is the main 
 benefits of Hugo. 
 
 Hence we have $pages = \bigcup_{S} \bigcup\_{I} gen(template_s, html\_{s,i})$. If we use 
 markdown (`md` file suffix) as a source format, and $S$ to denote a section(e.g. as a set of 
 pages) :
  
  $$ pages = \bigcup\_{S} \bigcup\_{I} gen(template\_s, md2html(md\_{s,i})) $$

What are the benefits? On one side, it comes down to separation of concerns. Separating a page 
between a template and pure content allows to delegate the production of content to specialists 
without technical skills. On the other side, the template factors in a whole section of the 
website, and is automatically woven into the content, hence there is also a productivity gain.

# Hugo
Hugo takes a root directory, with an expected structure, containing content with expected formats, and turns into to-be hosted website content. 

- the `content` directory contains the fixed content of the website to be generated
- any other directory will be in the resulted website, without modification
- any content `c` from the `content` directory is transformed into the corresponding content on the 
resulted website :
  - $page = gen(templates\(c), c)$
- the following template types are recognized by Hugo
  - list templates
    - home page, section, taxonomy, content view, rss 
  - single page templates
  - sitemap templates  
- Data can be passed at the according level through data templates
- A mechanism extending markdown syntax can be configured through `shortcode` templates
- Google Analytics, Disqus can be included through internal templates 
- A base template can be used as a template of templates, in which other templates (partial 
templates) are included (for DRY purposes)
 
## Templating mechanism
Given a `.md` content file, a `.html` file is derived by looking up for a `html` template, and 
helper `md` files. Specific rules depend on the type of content to be generated. Cf. Hugo 
documentation for the template lookup mechanism and order. 

Generation of final html is based on variables which are automatically evaluated by Hugo, such 
the page title, the creation date, etc. The list of such variables can be found in the 
documentation.

In fine, default templating, and overriden templating allow for DRY templating. This means that 
to reconstruct a page from a `.md` file, one may have to consult several sources of templates, in 
several directories, which can be a tradeoff.

Additionally weaving the dynamic aspects of the page into the fixed content requires learning 
another language and its syntax and system classes (pages, table of contents, etc.). This is not 
unlike for instance Office/VBA, which is the customization language for Office. However, without 
a dedicated IDE, it will remain painful to author, debug, and maintain such programs. As such 
Hugo is better fit for blogs, or any website where precisely the dynamic part is not often subject 
to maintenance (i.e. write once and forget).

# Issues encountered
- templates do not compose!
  - sounds obvious, there are not meant to be composed. However, one cannot take parts of a theme
   (say table of contents and pagination) and mix it with other parts of a theme (say taxonomy 
   and menu) easily.
- DSL has to be learnt, and no IDE to support it so far. Poor debugging and error feedback (limited
 to `printf`)
- the rules to associate templates to a given `.md` content are complex and require looking up a 
potentially large number of files (config.toml, data templates, terms template, localization, 
archetypes etc.)

# Tips
- menu in toml menu.main
- to add a menu, go to toml, menu.main, then also change index.html and add the corresponding term  ex. `<h4>{{ T "latestPost" }}</h4>`
  - or in the .md, add a menu : "main"!! makes sense
- tags, categories, i.e. taxonomies must have map singular/plural in toml file under `[taxonomies]`
- use weights in content matter (index.md) or post.md or toml to have ordering
  - for example here with menu="main"
- possibility to have directly templates for home/section/ or use a template's template i.e. baseof. cf. `https://gohugo.io/templates/base/`
- mathjax in hugo : some issues with more technical needs
   - multi line : us `\\\\\` instead of `\\`
   - accolade : use `spaces`
   
```latex
$$\begin{cases}
   view & = f_v(state) \\\\\
   actions & = f_a(state, events)
\end{cases} $$
```
