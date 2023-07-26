# Python Packaging and Deployment: State of Art (2023)

### by Bartek Ogryczak <[`bartek.fyi`](https://bartek.fyi/)>

[![CC BY-SA 4.0][cc-by-sa-shield]][cc-by-sa]

## Synopsis

This will be an opinionated, streamlined recap of the state of Python packaging, including hosting private packages, 
and Python services deployment. 

The subjects I intent to cover:
- various package types
- dealing with package dependencies
- various options for hosting private packages
- building and deploying Python services 

The subjects I will specifically not cover:
- Python 2, universal py2.py3 packages, etc. That ship has sailed a decade ago 
- Tools for standalone executables (such as PyInstaller & py2exe)

This is not meant to replace in depth sources like [Python Packaging User Guide](https://packaging.python.org/), 
but rather give you a quick overview of most relevant info. While my primary focus is on Python (micro)services 
in cloud, this should be useful for everyone.  

## Motivation

> _"In the face of ambiguity, refuse the temptation to guess.  
> There should be one — and preferably only one — obvious way to do it."_  
>                  — _["The Zen of Python"](https://peps.python.org/pep-0020/)_ 

I was recently asked for recommendation for best practices for build tools we should use company wide. 
To my surprise since the last time I took a deep dive into the subject, not only there was no consensus, 
but the number of specifications, build tools, and deployment managers has more than doubled.
[A new one](https://github.com/mitsuhiko/rye) has just been released as I was researching this subject. 
Thus the answer to this question has never been more complex and contentious.  

[![XKCD: Python Environment](https://imgs.xkcd.com/comics/python_environment.png)](https://xkcd.com/1987/)

Back in 2017 I wrote an extensive two part blog post on [Python packaging and releasing private code](
    https://www.eventbrite.com/engineering/packaging-and-releasing-private-python-code-pt-1/). 
A lot has improved since then, a lot of issues that I mentioned that would be solved in the near future
have already been implemented, some of the issues become moot. So while it _was_ decent resource back then,
it is obsolete now, and this document addresses that by being current, and attempting to be somewhat 
future-proof by describing aspirational goals set by current PEP and PyPA standards, the way things
_should_ be done, rather than then things often are done. That being said, even though these are 
aspirational goals, all of this is already fully implemented and works as described. 

## Contents

### * [Part 1: Packaging](packaging.md) 
### * Part 2: Specifying Dependencies _(coming soon)_
### * Part 3: Hosting Packages _(coming soon)_


--- 
[![CC BY-SA 4.0][cc-by-sa-image]][cc-by-sa] This work is licensed under a [Creative Commons Attribution-ShareAlike 4.0 International License][cc-by-sa].

[cc-by-sa]: http://creativecommons.org/licenses/by-sa/4.0/
[cc-by-sa-image]: https://licensebuttons.net/l/by-sa/4.0/88x31.png
[cc-by-sa-shield]: https://img.shields.io/badge/License-CC%20BY--SA%204.0-lightgrey.svg
