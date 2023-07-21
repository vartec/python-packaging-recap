# Wheels Compatibility Tags
Wheels are _distribution packages_, as such their content is unzipped _as is_ into the `site-packages` directory. Wheel format started as an official PSF [PEP-427](https://peps.python.org/pep-0427/), but is [currently maintained by PyPA](https://packaging.python.org/en/latest/specifications/binary-distribution-format/). 

A wheel is a standard ZIP-format archive with a custom extension `.whl`, following very specific and strict naming structure.
Theoretically `.whl` could be used without unzipping, but it's discouraged and not a common practice. 

# Understanding Wheel File Name

All Wheel file names must conform to this template. This will come into play later, when we talk about package repositories. 

```{distribution}-{version}(-{build tag})?-{python tag}-{abi tag}-{platform tag}.whl.```
 
Where:
- `distribution` is the package name;
- `version` is a valid version, `[N!]N(.N)*[{a|b|rc}N][.postN][.devN]` as specified by the [PEP-440](https://peps.python.org/pep-0440/);
- `build tag` is completely optional, rarely use it, though for internal packages you could use it for the CI build number;
- `python tag` identifies compatible Python version, two letters followed by digits, first of which indicates major version:
    - `py` - universal Python, indicates that package works with any Python implementation, indicative of pure Python packages;
    - `cp` - CPython, the standard Python version implemented in C;
    - `pp` - PyPy, a JIT-compiled implementation of Python;
    - `ip` - IronPython, C#.NET implementation of Python, effectively defunct;
    - `jy` - Jython, JVM implementation of Python, completely defunct; 
- `abi tag` identifies the application binary interface used (or C API in case of CPython):
    - `none` - a pure Python package, does not depend on any ABI; 
    - `abi3` - CPython 3 stable ABI, does not depend on any specific minor version;
    - a specific _python tag_, as defined in previous section, followed by one of flags
       - `d` - compiled with debug symbols;
       - `m` - _[deprecated]_ compiled with PyMalloc allocator, this flag is no longer used since Python 3.8, as it's the default;
       - `u` - _[deprecated]_ compiled with wide (32-bit) Unicode, this flag is no longer used since Python 3.3, as it's the default;  
    - example: `cp32mu` - CPython 3.2 compiled with PyMalloc and USC4 Unicode.  
- `platform tag` - indicates software platform (typically operating system or family of systems), in most cases it's normalized output of `sysconfig.get_platform()`, which few exceptions:
  - examples: 
    - `linux_x86_64` - any Linux on x86_64; 
    - `macosx_11_0_arm64` - MacOS 11 or newer on Apple Silicon; 
    - `win_amd64` - 64-bit Windows on x86_64; 
  - other special cases:
    - `any` - not platform dependent, indicative of pure Python packages; 
    - `manylinux_{X}_{Y}_{ARCH}` (example `manylinux_2_17_aarch64`) - Linux with version of _glibc_ >= X.Y (see [PEP-600](https://peps.python.org/pep-0600/)). 
    - `manylinux_{2010|2014}_{ARCH}` _[deprecated]_ - (example `manylinux_2014_x86_64`) - Legacy _manylinux_ specifier, each had a corresponding spec, [PEP 571](https://peps.python.org/pep-0571/) (manylinux2010), [PEP 599](https://peps.python.org/pep-0599/) (manylinux2014);  
    - `manylinux1` _[deprecated]_ - even older legacy specifier, the very first _manylinux_. Defined in [PEP-513](https://peps.python.org/pep-0513/);
    - `musllinux_{X}_{Y}_{ARCH}` (example `musllinux_1_1_x86_64`)  - Linux using _musl_ >= X.Y as its clib (for example Alpine Linux);  

> **Note**  
> Any of the tags can be combined using dot `"."`, meaning that package is compatible all of these tags. For example `py2.py3` indicates a package compatible with Python 2 and Python 3.  
> Use sparingly, as using it on multiple tags creates cartesian product of all tag combinations, which can quickly get out of hand

Some real life examples:

- `numpy-1.25.1-cp311-cp311-musllinux_1_1_x86_64.whl` - NumPy 1.24.1 compiled for CPython 3.11 on (Alpine) Linux with musl >= 1.1 on x86_64 hardware;
- `Django-4.2.3-py3-none-any.whl` - Django 4.2.3. It's a pure Python package, hence it's compatible with any Python 3. 

