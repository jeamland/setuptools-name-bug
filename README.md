# A demonstration of a bug in setuptools

## What's this?

Python's [setuptools](https://github.com/pypa/setuptools) project is an awesome thing and is only getting awesomer over time. That said, I found a bug in it. It's not a huge one, but it's an irritant.

The bug goes like this:

- You have a Python project using `pyproject.toml`
- You are using the ability to specify the version as a dynamic element
- You are specifying the source of the dynamic version to be an attribute, using configuration in `pyproject.toml`
- You run `python setup.py --name`

The result looks a bit like this:

```console
$ python setup.py --name
/Users/benno/.pyenv/versions/setuptools-name-bug/lib/python3.10/site-packages/setuptools/config/pyprojecttoml.py:108: _BetaConfiguration: Support for `[tool.setuptools]` in `pyproject.toml` is still *beta*.
  warnings.warn(msg, _BetaConfiguration)
Traceback (most recent call last):
  File "/Users/benno/work/setuptools-name-bug/setup.py", line 3, in <module>
    setup()
  File "/Users/benno/.pyenv/versions/setuptools-name-bug/lib/python3.10/site-packages/setuptools/__init__.py", line 87, in setup
    return distutils.core.setup(**attrs)
  File "/Users/benno/.pyenv/versions/setuptools-name-bug/lib/python3.10/site-packages/setuptools/_distutils/core.py", line 170, in setup
    ok = dist.parse_command_line()
  File "/Users/benno/.pyenv/versions/setuptools-name-bug/lib/python3.10/site-packages/setuptools/_distutils/dist.py", line 471, in parse_command_line
    args = parser.getopt(args=self.script_args, object=self)
  File "/Users/benno/.pyenv/versions/setuptools-name-bug/lib/python3.10/site-packages/setuptools/_distutils/fancy_getopt.py", line 274, in getopt
    val = getattr(object, attr, 0) + 1
TypeError: can only concatenate str (not "int") to str
```

## What's going on?

The `name` attribute of a setuptools/distutils `Distribution` object is not actually meant to be the distribution name. It's meant to be a value indicating whether the command-line invocation of setup wants to see the name attribute. The name itself is stored in a metadata object that is kept as its own attribute of the `Distribution` object. In commit [1203ee23c](https://github.com/pypa/setuptools/commit/1203ee23c979175b0f9c7e4eb3854e19df95e3b2) a change was made to enable automatic configuration discovery and in that there ends up being a codepath that does this:

```python
            self.dist.metadata.name = name
            self.dist.name = name
```

You can see the method in question [here](https://github.com/pypa/setuptools/blob/main/setuptools/discovery.py#L468).

The first line seems to be correct, but the second seems to be erroneous and ends up sticking the distribution name (a string) on to the `Distribution` object which expects it to be an integer later on in the command line processing gubbins.

This fix appears to be to simply remove the second line. This has been filed as [issue 3545](https://github.com/pypa/setuptools/issues/3545) and a change submitted as [PR 3547](https://github.com/pypa/setuptools/pull/3547).
