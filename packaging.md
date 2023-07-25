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

Most likely you package has dependencies, you can also specify optional dependencies (also known as _extras_). 

```toml
[project]
# ...
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
# ...
requires-python = ">=3"
```


### Descriptive Fields 

These fields are not required to _build_ the package, but are must have for any public package, and I'd still highly recommend them even 
for internal, private packages.

- `description` - a summary description of the package, preferably a single line;
- `license` - can be either name of a license in [SPDX standard](https://spdx.org/licenses/) or a relative path to license file as `{file: <path_of_license_file>}`. 
   Note that in `setup.(py|cfg)` these were two different fields, `license_file` and `license` respectively. 
- `readme` - relative path to the README file.
- `authors` & `maintainers` - lists of author and maintainers with at least a name and a email, eg. `{name = "Dev Eloper", email = "dev@example.com"}`.
- `keywords` - array of string of keywords for search in indexes such as PyPI.
- `project.urls` - a list of `key = URL`, there is no predefined list of keys you need to have, but a bare minimum would be a `homepage`,
   which used to be a top level `url` field in `setup.(py|cfg)`. I'd strongly recommend also adding at least `source` and `documentation`.
- `classifiers` - a list of strings classifying the project by it's maturity, environment, intended audience, supported OS, programming language, topic, etc.
   PyPI maintains [a complete list of classifiers](https://pypi.org/classifiers/). 

Example: 
```toml
[project]
name = "foo"
version = "0.9.8"
description = "A Foo for a Bar and Baz"
readme = "README.md"
requires-python = ">=3.11"
license = {file = "LICENSE"}
keywords = ["foo", "bar", "baz", "foo bar"]
authors = [
  {name = "Dev Eloper", email = "deve@example.com"},
  {name = "Señor Dev", email = "sdev@example.com"},
]
maintainers = [
  {name = "Mai Tainer", email = "mai@example.com"}
]
classifiers = [
  "Development Status :: 4 - Beta",
  "Programming Language :: Python",
]
dependencies = [
  "Django ~= 4.2",
]

[project.optional-dependencies]
test = [
  "pytest",
  "pytest-cov[all]",
]

[project.urls]
homepage = "https://foo.example.com"
documentation = "https://example.readthedocs.org/"
source = "https://github.com/ex/ample/"
```

**pro-tip**: If a package is private and you want to prevent in being accidentally uploaded to PyPI, adding any classifier starting with `Private ::` 
will ensure PyPI automatically rejects it:
```toml
[project]
# ...
classifiers = [
    "Private :: Do Not Upload",
]
```

### Single-Sourcing the Package Version
It's a common practice to have package version stored in `package/version.py` or `package/__init__.py`, which would then be available as 
`package.__version__` Python code. This creates a chicken-egg problem for the packaging tools, as you need to know the version of the package,
_before_ the package available, thus before `package.__version__` could imported. Having the version in one single source of truth 
is known as _single-sourcing the package version_(https://packaging.python.org/en/latest/guides/single-sourcing-package-version/).  
In the legacy `setup.py` this required a gnarly workaround, and while there are many implementations, they all basically boil down
to manually reading the specified file, than either using regex to extract the version, `exec`-ing the whole file, or outright
having version stored in another plain-text file (which adds even more problems). None of these are great. 

Since the release 61.0 of _setuptools_ same can be done significantly easier directly in `pyproject.toml`:

```toml
[project]
name = "package"
dynamic = ["version"]

[tool.setuptools.dynamic]
version = {attr = "package.__version__"}
```

Note, that the section `[tool.setuptools.dynamic]` is _setuptools_ specific, as such won't work with other build backends. 

### Migrating `setup.py` and/or `setup.cfg` to `pyproject.toml`

> **Warning**  
> This is slightly controversial topic in Python community, and there still are devs who feel like `pyproject.toml`
> has been forced upon them for no reason. I won't get into the details or take sides. What I am presenting 
> in this document are the best practices as described in a PSF approved PEPs, and PyPA's packaging guidelines. 
 
As I've mentioned earlier, as of 2023, `pyproject.toml` can fully replace both `setup.py` and `setup.cfg`.
Unfortunately legacy setuptools does not make a clear distinction between which `setup()` parameters in `setup.py`, 
or which keys in `setup.cfg` are setuptools specific, but there's [a helpful list in setuptools documentation](
    https://setuptools.pypa.io/en/latest/userguide/pyproject_config.html#setuptools-specific-configuration). 

The migration from `setup.py` might be tedious, but not complicated, you just take all  parameters that are not setuptools 
specific from `setup()` and convert them into fields of `[project]` section in `pyproject.toml`. 
Some fields may require renaming:
    - `install_requires` becomes `dependencies`
    - `extras_require` become `optional-dependencies`
    - `url` becomes `homepage` in `[project.urls]`;
    - `author` & `author_email`, as well as `maintainer` & `maintainer_email` are rolled into `authors` and `maintainers` 
       list, containing both name and email;
    - `entry_points` become `[project.scripts]`
    - `python_requires` becomes `requires-python`
The `setup()` parameters that _are_ setuptools specific got into `[tool.setuptools]` section or its subsections.

The above also applies to migrating from `setup.cfg`, where these keys are stored in `[metadata]` and `[option]` sections.

Regarding `packages=find_packages...`, for [standard project layouts, flat-layout and src-layout](
    https://packaging.python.org/en/latest/discussions/src-layout-vs-flat-layout/), 
setuptools does a good job as auto-discovering, so in most cases it might be no longer necessary. 

Abridged example, a legacy `setup.py`:

```python
from setuptools import setup

setup(
    name = 'foo',
    version = '0.9.8',
    description = 'A Foo for a Bar and Baz',
    python_requires = '>=3.11'
    author = 'Dev Eloper <deve@example.com>',
    maintainer = 'Mai Tainer <mai@example.com>',
    packages = find_packages(include=['foo', 'foo.*']), 
    classifiers = [
        "Development Status :: 4 - Beta",
        "Programming Language :: Python",
    ],
    install_requires = [
       'Django ~= 4.2',
    ],
    extras_require = {
       'test': ['pytest', 'pytest-cov[all]'],
    },
    url = 'https://foo.example.com',
    project_urls={
        'documentation': 'https://example.readthedocs.org/',
        'source': 'https://github.com/ex/ample/',
    },
)
```

Converted to `pyproject.toml` becomes:

```toml
[project]
name = "foo"
version = "0.9.8"
description = "A Foo for a Bar and Baz"
requires-python = ">=3.11"
authors = [
  {name = "Dev Eloper", email = "deve@example.com"},
]
maintainers = [
  {name = "Mai Tainer", email = "mai@example.com"},
]
classifiers = [
  "Development Status :: 4 - Beta",
  "Programming Language :: Python",
]
dependencies = [
  "Django ~= 4.2",
]
[project.optional-dependencies]
test = [
  "pytest",
  "pytest-cov[all]",
]

[project.urls]
homepage = "https://foo.example.com"
documentation = "https://example.readthedocs.org/"
source = "https://github.com/ex/ample/"
```

But wait! — you say — there's a lot more stuff in my `setup.cfg`, can I migrate that too?

Yes you can, for most of the tools `[<toolname>]` from `setup.cfg` becomes `tool.<toolname>`.  
Note, that there are some tools that refuse to support `pyproject.toml`, thus you either 
need [a workaround](https://github.com/csachs/pyproject-flake8) or switch to [different tools](https://black.readthedocs.io/). 

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
