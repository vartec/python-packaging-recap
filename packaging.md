# Part 1: Packaging

This part covers creating Python packages. 

## Package Types

### Source Distribution aka `sdist`
`sdist` or a _source distribution_ is basically just a compressed archive (typically a `.tar.gz`) of the sources 
with a little additional metadata. It's not directly installable, requires a _build_ step before it's installed. 
In case of pure Python packages this could be good enough, but definitely falls short in case of Python bindings 
for compiled libraries. Installing an _sdist_ will typically build a Wheel locally, which then gets installed 
directly. Building requires all of the build dependencies (tools, development version of libraries, etc.), which 
might be non-trivial, and generally not something you'd want to depend on as part of your deployment strategy. 
Also, for some libraries it takes quite a while to compile, you wouldn't want your CI jobs waiting 20 minutes for 
let's say NumPy or SciPy to compile before it can even start running tests. 

While there is no formal specification for legacy setup.py-style sdist, for current compliant pyproject.toml-style 
sdists the format is specified in [PEP-517](https://peps.python.org/pep-0517/)

> **Note**  
> While there are many [formats supported by _setuptools_](https://docs.python.org/3/distutils/sourcedist.html), 
> you might be tempted to choose `.tar.xz` or `.tar.bz2` for significantly better compression ratios, 
> however keep in mind that for public packages, per [PEP-527](https://peps.python.org/pep-0527/), 
> only `.tar.gz` and `.zip` are allowed on PyPI, while [PEP-517](https://peps.python.org/pep-0517/) amongst other things
> formalizes _sdist_ file format as gzipped tar, which is further reinforced by formalized naming scheme 
> [PEP-625](https://peps.python.org/pep-0625/) accepting `.tar.gz` extensions only. 

### Wheels
Wheels are _distribution packages_, as such their content is unzipped _as is_ into the `site-packages` directory. Wheel format started as an official PSF [PEP-427](https://peps.python.org/pep-0427/), but is [currently maintained by PyPA](https://packaging.python.org/en/latest/specifications/binary-distribution-format/). 

A wheel is a standard ZIP-format archive with a custom extension `.whl`, following very specific and strict naming structure.

```{distribution}-{version}(-{build tag})?-{python tag}-{abi tag}-{platform tag}.whl.```

All Wheel file names must conform to this template. This will come into play later, when we talk about package repositories.  
To learn about meaning of each of these tags go to [wheels' tags](wheels-tags.md) sub-page. 

Some real life examples:

- `numpy-1.25.1-cp311-cp311-musllinux_1_1_x86_64.whl` - NumPy 1.24.1 compiled for CPython 3.11 on (Alpine) Linux with musl >= 1.1 on x86_64 hardware;
- `Django-4.2.3-py3-none-any.whl` - Django 4.2.3. It's a pure Python package, hence it's compatible with any Python 3. 

### Eggs _[deprecated]_

> **Note**  
> This format has been deprecated, most tools no longer support creating it at all, and it's scheduled to be disallowed 
> in PyPI per [PEP-715](https://peps.python.org/pep-0715/).   
> I'm including it because you might still encounter some
> legacy packages that use Eggs. Also, because there are outliers like for example PySpark, which [supports Eggs, but not
> Wheels](https://spark.apache.org/docs/latest/api/python/user_guide/python_packaging.html) in its package management. 

This is a legacy distribution format, that was obsoleted with introduction of Wheels. Just as Wheels, it's a ZIP-format 
archive, this time with custom extension `.egg`. Unlike Wheel format, which has formal specification, Egg was an ad-hoc 
format added by `setuptools`. 

### Other Formats

There are a number of other Python distribution package formats, however they are non-standard, none of the package formats listed below are compatible with standard installers. As such they aren't used outside of their specific niches, so I'll only mention them briefly:
- [Conda](https://docs.conda.io/) packages (`.conda` or `.tar.bz2`), a tarball of `site-packages/{package}/` and Conda specific metadata in `info/`. Conda is packaging, dependency & environnement management tool popular in Data Science applications. 
You can tell Conda package apart from _sdist_ packages, as sdist is `{package}-{version}.tar.gz`, while Conda packages are `{package}-{version}-{build_tag}.tar.bz2`;
- [PEX](https://github.com/pantsbuild/pex) (`.pex`), Python EXecutable, a full virtualenv distributed as a self-contained package. Used, amongst other DE/ML tools, by PySpark;
- `.rpm`, Linux package for RedHat based Linux distributions; 
- `.dmg`, `.msi`, OS dependent installers for Windows and MacOS respectively;
- Jupyter Notebooks (`.ipynb`) - not sure if you can call it a Python package, but it is a way of sharing Python code. A Jupyter Notebook is a JSON format which combines its metadata, with one or more segments (called _cells_ in Jypyter lingo), each of which can be one either markdown, code (traditionally Python or R, but more languages are added), or rich media output data which can be used for visualization using various image formats. 

## Package Specification

### One `pyproject.toml` to Rule Them All

You can have whole specification of your package in just one file - `pyproject.toml`. It can, with some caveats, replace both `setup.py` and `setup.cfg`.
Project specification goes into `[project]` section and `[project.*]` sub-sections. 

### Minimum Set of Fields

There are only two project specification fields that are absolutely required to build a package.
```toml
[project]
name = "myexamplepackage"  # the name of the package
version = "1.2.3"  # a PEP-440 compliant version string
```

Most likely you package has dependencies, you can also specify optional dependencies. 

```toml
[project]
⋮
dependencies = [   # is setup.py this was called "install_requires"
  "required_dependency"
]

[project.optional-dependencies]  # in setup.py this was called "extras_require"
dev = ["ipdb"]  # name of the extra followed by list of requirements 
test = ["pytest"]  
```

Unless you're creating a universal Python2 / Python3 package, which in 2023 is unlikely, you need to specify minimum Python version:
```toml
[project]
⋮
requires-python = ">=3"
```

If a package is private and you want to prevent in being accidentally uploaded to PyPI, adding any classifier starting with `Private ::` 
will ensure PyPI automatically rejects it:
```toml
[project]
⋮
classifiers = [
    Private :: Do Not Upload,
]
```

### Descriptive Fields 




## Building Packages

### Using build _[highly recommended]_

[`build`](https://pypa-build.readthedocs.io/) is a simple implementation of build frontend as described in [PEP-517](https://peps.python.org/pep-0517/) It's created and maintained by the the [Python Packaging Authority (PyPA)](https://www.pypa.io/), which recommends using this rather than `pip` for modern PEP-517 conforming packages, meaning packages which have `pyproject.toml` that includes `[build-system]` section. 

_Build_ by default will build both the _sdist_ and and then the _wheel_ from that _sdist_. Resulting package files with got to `${srcdir}/dist/` unless otherwise specified. Unless specifically disabled with `--no-isolation`, _build_ sets up and uses _venv_ to isolate the build from system libraries. 
```shell
python3 -m build # build sdist and wheel in current directory, put them in ./dist/
python3 -m build ${srcdir} --outdir /tmp/wheels # build sdist and wheel from ${srcdir}, put them in /tmp/wheels
```

### Using pip 

[`pip`](https://pip.pypa.io/en/stable/user_guide/) is the Python package installer, also created and maintained by the PyPA. 

Remember do add trailing slash to the `srcdir`, otherwise _pip_ will assume it's a package name and attempt to download it from PyPI.
You can also use `.` as `srcdir` for current directory. 

`pip wheel` will build the package itself _and_ all the required dependencies, you can force building _only_ the package itself by passing `--no-deps` option.

```shell
python3 -m pip wheel ${srcdir}  # creates wheels in current directory
python3 -m pip wheel ${srcdir} --wheel-dir=${whldir}  # creates wheels in specified directory
python3 -m pip wheel ${srcdir} --no-deps  # creates only the wheel for the package itself, not for dependencies 
```

_Pip_ does not support building _sdist_ packages. 

### Using setuptools Directly _[deprecated]_

[_Setuptools_](https://setuptools.pypa.io/) is a library facilitating Python packaging, it was created as a replacement for disutils. 
It's still the primary choice for Python packaging. However it's now considered a _build backend_, as such should not be invoked directly, 
but rather through a build frontend, such as `build` mentioned above. 

Packages will be added to `./dist/`

```shell
python3 setup.py sdist  # builds sdist
python3 setup.py sdist --formats=gztar  # build sdist forcing format to be .tar.gz
python3 setup.py bdist_wheel  # builds wheel
```
> **Note**  
> While [_setuptools_](https://setuptools.pypa.io/) are not deprecated, and in fact are actively maintained, using them as build system
> directly is. You should [configure _setuptools_ in your `pyproject.toml`
> ](https://setuptools.pypa.io/en/latest/userguide/pyproject_config.html#setuptools-specific-configuration) instead.  
> In many cases both `build` and `pip` will invoke _setuptools_ underneath the hood.    


###  Build Backends

There are now number of alternative build backends, aside from setuptools. Keep in mind that the same logic
applies to all, they should not be invoked _directly_, but rather used through `build` frontend configured in 
`[build-system]` section of `pyproject.toml`. 

This is not an exhaustive list, these are just the most commonly used backends. 

#### setuptools
`build` config for _setuptools_:

```toml
[build-system]
requires = ["setuptools>=61.0"]
build-backend = "setuptools.build_meta"
```

#### Flit 
[_Flit_](https://flit.pypa.io/) is the proof of concept and de facto reference implementation for [PEP-518](https://peps.python.org/pep-0518/) and [PEP-517](https://peps.python.org/pep-0517/). Aims to be as streamlined as possible, especially for simple cases like pure Python packages. 

```toml
[build-system]
requires = ["flit_core>=3.4"]
build-backend = "flit_core.buildapi"
```

#### PDM 
[_PDM_](https://pdm.fming.dev/) is modern Python package and dependency manager inspired in NPM, with a goal of supporting the latest PEP standards. 
[`pdm-backend`](https://pdm-backend.fming.dev/) is its build backend, which can be used with PDM or as a backend for `build`.

```toml
[build-system]
requires = ["pdm-backend"]
build-backend = "pdm.backend"
```

#### Hatchling 
_Hatchling_ is a backend used by [_Hatch_](https://hatch.pypa.io/), another Python project manager. Again, can be used standalone as `build` backend.
```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```


#### Poetry 
[_Poetry_](https://python-poetry.org/) is yet another package and dependency manager, quite a popular one, if any dependency manager had chance 
at replacing `pip` it would be this one. Poetry is the project that was first to introduce `pyproject.toml`. Its build backed is `poetry-core`.
```toml
[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```
