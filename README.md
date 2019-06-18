# Python Fixes

These fixes do not necessarily need to be patched into Python proper. Users _can_ create a helper module or subclass `ZipFile`. But directly or indirectly related issues on the Python issue tracker have been open and unresolved for _years_.

## NTFS Junction Points

1. `os.walk()` ignores broken NTFS junction points. Initially, this _sounds_ like a good idea, but users should be able to validate all paths with their own code.
2. `ntpath.islink()` returns `False` for both broken and working NTFS junction points because this function does not support NTFS junction points.

### Notes

#### pathlib

`pathlib` may require its own set of fixes to properly handle NTFS junction points.

#### ntpath.isdir() vs. genericpath.isdir()

To test whether a junction point is a valid directory, users should use `genericpath.isdir()`. According to the comments in `ntpath.py`, the core maintainers believe `genericpath.isdir()` is "overkill" for Windows; it _probably_ is for most use cases. Perhaps someone profiled that at some point? Hopefully...?

```python
try:
    # The genericpath.isdir implementation uses os.stat and checks the mode
    # attribute to tell whether or not the path is a directory.
    # This is overkill on Windows - just pass the path to GetFileAttributes
    # and check the attribute from there.
    from nt import _isdir as isdir
except ImportError:
    # Use genericpath.isdir as imported above.
    pass
```

If users know they will be working with NTFS junction points, they should know how these functions differ:

- `ntpath.isdir()` will return `True` for both broken and working NTFS junction points.
- `genericpath.isdir()` will return `False` for broken junction points and `True` for working junction points.


## ZipFile

> [Issue 6839](https://bugs.python.org/issue6839): ZipFile.open() raises BadZipFile exception when opening ZIP file

The following file name encoding test is the origin of that exception.

```python
            if zinfo.flag_bits & 0x800:
                # UTF-8 filename
                fname_str = fname.decode("utf-8")
            else:
                fname_str = fname.decode("cp437")

            if fname_str != zinfo.orig_filename:
                raise BadZipFile(
                    'File name in directory %r and header %r differ.'
                    % (zinfo.orig_filename, fname))
```

This standalone test is highly opinionated and should not be executed at the module scope. Users should determine whether they need to run this test in their own code. In the vast majority of use cases, it is unlikely.
