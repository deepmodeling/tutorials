# Writing Tips

Hello volunteers, this docs tells you how to write articles for DeepModeling tutorials.

You can just follow 2 steps:
1. Write in markdown format and put it into proper directories.
2. Change index.rst to show your doc.

## Write in Markdown and Put into proper directories.


1. You should learn how to write a markdown document. It is quite easy!

Here we recommend you 2 website:

* [English Tutorial](https://www.markdownguide.org/)

* [中文版教程](https://markdown.com.cn/)

2. You should know the proper directories.

Our Github Address is: https://github.com/deepmodeling/tutorials

All doc is in: "source" directories. According to your doc's purpose, your doc can be put into 4 directories in "source":
* Tutorials: Telling beginners how to run Deepmodeling projects.
* Casestudies: Some case telling people how to use Deepmodeling projects.
* Resources: Other resources for learning.
* QA: Some questions and answers.

After that, you should find the proper directories and put your docs.

For example, if you write a "methane.md" for case study, you can put it into "/source/CaseStudies/Gas-phase".

## Change indexs.rst to show your doc.

Then you should change the index.rst to show your doc.

You can learn rst format here: [reStructuredText](https://docutils.sourceforge.io/rst.html)

In short, you can simply change index.rst in your parent directories.

For example, if you put "methane.md" into "/source/CaseStudies/Gas-phase", you can find and change "index.rst" in "source/CaseStudies/Gas-phase". All you should do is imitating such file. I believe you can do it!




---

If you want to learn more detailed information about how to build this website, you can check this:

* [Sphinx + Read the Docs(Chinese)](https://zhuanlan.zhihu.com/p/264647009)

* [Sphinx](https://www.sphinx-doc.org/en/master/)

* [readthedocs](https://link.zhihu.com/?target=https%3A//readthedocs.org/)

* [reStructuredText](https://docutils.sourceforge.io/rst.html)
