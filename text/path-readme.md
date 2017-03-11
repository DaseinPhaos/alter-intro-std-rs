% std::path

This module essentially provides two types `PathBuf` and `Path` (akin to `String` and `str`) for cross-platform path manipulation.

To create an immutable path, one can create `Path` slice directly from a `str` slice.

To modify paths, one use `PathBuf`.

# Rules o Paths

A path consists of a sequence of "components", which roughly correspond to the substrings between path separators.

Specifically, current directory component is represented by a `.` character, while root directory component is represented as a prefixed separator (on Windows, a disk-prefix, e.g. `C:\\`, might also be included).

By default, when building a path two normalizations are performed:

- Repeated separators would be ignored, thus `a/b` and `a//b` both have components `a` and `b`.
- Component `.` not at the head ot the path would be ignored, thus `././a/./b/.` would get normalized to `./a/b`.

# Component

A single component of the path is represented by

```ignore
pub enum Component<'a> {
    Prefix(PrefixComponent<'a>), // A Windows path prefix
    RootDir, // appears after any prefix, before anything else
    CurDir, // i.e. `.`
    ParentDir, // i.e. `..`
    Normal(&'a OsStr), // a normal component
}
```

It can be transformed into an `&OsStr` with method `fn as_os_str(self) -> ...`.

Note that it implements `Copy`, `Clone`, and `AsRef<OsStr>`.


# Path

```ignore
pub struct Path {
    inner: OsStr,
}
```

Represents a slice of a path.

To construct a slice of it:

```ignore
impl Path {
    pub fn new<S: AsRef<OsStr> + ?Sized>(s: &S) -> &Path {
        unsafe { std::mem::transmute::(s.as_ref()) }
    }
}
```

To transform it as a `&OsStr` or to a `&str` or loosely to a `String` or to an owned `PathBuf`:

```ignore
impl Path {
    pub fn as_os_str(&self) -> &OsStr {
        &self.inner
    }

    pub fn to_str(&self) -> Option<&str> {
        self.inner.to_str()
    }

    pub fn to_string_lossy(&self) -> Cow<str> {
        self.inner.to_string_lossy()
    }

    pub fn to_path_buf(&self) -> PathBuf {
        PathBuf::from(self.inner.to_os_string())
    }
}
```

To determine if it is absolute (i.e. independent of the current working directory) or relative (i.e. not absolute):

```ignore
impl Path {
    pub fn is_absolute(&self) -> bool {/*...*/}

    pub fn is_relative(&self) -> bool {
        !self.is_absolute()
    }
}
```

To produce a sequenced iterator over the components of the path:

```ignore
impl Path {
    pub fn components(&self) -> Components {/*...*/}
}
```

To determine whether a path has a root component:

```ignore
impl Path {
    pub fn has_root(&self) -> bool {
        self.components().has_root()
    }
}
```

To get the parent path (if any):

```ignore
impl Path {
    pub fn parent(&self) -> Option<&Path> {
        let mut comps = self.components();
        let tail = comps.next_back();
        tail.and_then(|p| {
            match p {
                Component::Normal(_) |
                Component::CurDir |
                Component::ParentDir => Some(comps.as_path()),
                _ => None,
            }
        })
    }
}
```

To get the final component of the path. If the path terminates in `..`, this method returns `None`:

```ignore
impl Path {
    pub fn file_name(&self) -> Option<&OsStr> {
        self.components().next_back().and_then(|p| {
            match p {
                Component::Normal(p) => Some(p.as_ref()),
                _ => None,
            }
        })
    }
}
```

To strip out the `prefix` part of the path:

```ignore
impl Path {
    pub fn strip_prefix<'a, P>(&'a self, prefix: &'a P) -> Result<'a Path, StripPrefixError>
        where P: ?Sized + AsRef<Path>
    {
        self._strip_prefix(prefix.as_ref())
    }

    fn _strip_prefix<'a>(&'a self, prefix: &'a Path) -> Result<'a Path, StripPrefixError> {
        iter_after(self.components(), prefix.components())
            .map(|c| c.as_path())
            .ok_or(StripPrefixError(()))
    }
}

fn iter_after<A, I, J>(mut iter: I, mut prefix: J) -> Option<I>
    where I: Iterator<Item=A> + Clone,
          J: Iterator<Item=A>,
          A: PartialEq
{
    loop {
        let mut inext = iter.clone();
        match (inext.next(), prefix.next()) {
            (Some(ref x), Some(ref y)) if x == y => (),
            (Some(_), Some(_)) => return None,
            (Some(_), None) |
            (None, None) => return Some(iter),
            (None, Some(_)) => return None,
        }
        iter = inext;
    }
}
```

To find if the path starts with a prefix or ends with a suffix:

```ignore
impl Path {
    pub fn starts_with<P: AsRef<Path>>(&self, base: P) -> bool {
        self._starts_with(base.as_ref())
    }

    fn _starts_with(&self, base: &Path) -> bool {
        iter_after(self.components(), base.components()).is_some()
    }

    pub fn ends_with<P: AsRef<Path>>(&self, child: P) -> bool {
        self._ends_with(child.as_ref())
    }

    fn _ends_with(&self, child: &Path) -> bool {
        iter_after(self.components.rev(), child.components().rev()).is_some()
    }
}
```

To get the extension or non-extension part of the file name, we have:

```ignore
impl Path {
    pub fn file_stem(&self) -> Option<&OsStr> {
        self.file_name().map(split_file_at_dot).and_then(|(before, after)| before.or(after))
    }

    pub fn extension(&self) -> Option<&OsStr> {
        self.file_name().map(split_file_at_dot).and_then(|(before, after)| before.and(after))
    }
}

fn split_file_at_dot(file: &OsStr) -> (Option<&OsStr>, Option<&OsStr>) {
    unsafe {
        if os_str_as_u8_slice(file) == b".." {
            return (Some(file), None);
        }

        let mut iter = os_str_as_u8_slice(file).rsplitn(2, |b| *b == b'.');
        let after = iter.next();
        let before = iter.next();
        if before == Some(b"") {
            // The file starts with a `.`
            (Some(file), None)
        } else {
            (before.map(|s| u8_slice_as_os_str(s)),
             after.map(|s| u8_slice_as_os_str(s)))
        }
    }
}
```

To create a new owned path by joining two paths together:

```ignore
impl Path {
    pub fn join<P AsRef<Path>>(&self, path: P) -> PathBuf {
        self._join(path.as_ref())
    }

    fn _join(&self, path: &Path) -> PathBuf {
        let mut buf = self.to_path_buf();
        buf.push(path);
        buf
    }
}
```

To create a new owned path, but replace the current filename with `file_name`, or replace the current extension with `extension`:

```ignore
impl Path {
    pub fn with_file_name<S: AsRef<OsStr>>(&self, file_name: S) -> PathBuf {
        self._with_file_name(file_name.as_ref())
    }

    fn _with_self_name(&self, file_name: &OsStr) -> PathBuf {
        let mut buf = self.to_path_buf();
        buf.set_file_name(file_name);
        buf
    }

    pub fn with_extension<S: AsRef<OsStr>>(&self, extension: S) -> PathBuf {
        self._with_extension(extension.as_ref())
    }

    fn _with_extension(&self, extension: &OsStr) -> PathBuf {
        let mut buf = self.to_path_buf();
        buf.set_extension(extension);
        buf
    }
}
```

To create an iterator over the components of the path, viewing them as raw `OsStr` slices:

```ignore
impl Path {
    pub fn iter(&self) -> Iter {/*...*/}
}
```

To return a safely displayable path (this method is necessary because a path may contain non-Unicode data):

```ignore
impl Path {
    pub fn display(&self) -> Display {/*...*/}
}
```

To interact with the file system:

```ignore
impl Path {
    pub fn metadata(&self) -> io::Result<fs::Metadata> {
        fs::metadata(self)
    }

    pub fn symlink_metadata(&self) -> io::Result<fs::Metadata> {
        fs::symlink_metadata(self)
    }

    pub fn canonicalize(&self) -> io::Result<PathBuf> {
        fs::canonicalize(self)
    }

    pub fn read_link(&self) -> io::Result<PathBuf> {
        fs::read_link(self)
    }

    pub fn read_dir(&self) -> io::Result<fs::ReadDir> {
        fs::read_dir(self)
    }

    pub fn exists(&self) -> bool {
        fs::metadata(self).is_ok()
    }

    pub fn is_file(&self) -> bool {
        fs::metadata(self).map(|m| m.is_file()).unwrap_or(false)
    }

    pub fn is_dir(&self) -> bool {
        fs::metadata(self).map(|m| m.is_dir()).unwrap_or(false)
    }
}
```

Finally, note that it implements `ToOwned<Owned=PathBuf`, `AsRef<OsStr>`, `AsRef<Path>`, `Debug`, `PartialEq`, `Eq`, `PartialOrd`, `Ord`, `Hash`.

