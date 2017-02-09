% std::env

> https://doc.rust-lang.org/std/env/

This module defines some functions inspecting and manipulating the environment of the running process.

The "environment" here includes aspects such as environment argments, current directory, command line arguments and so on.

It also contains a submodule `const` which defines some constant values specifically associated with the current building target.

Let first take a look at some functions.

# Command Line Arguments

```ignore
use sys;

// ...

pub fn args_os() -> ArgsOs {
    ArgsOs{ inner: sys::args::args() }
}

pub fn args() -> Args {
    Args{ inner: args_os() }
}
```

These two functions both return an iterator over the arguments the program started with (these arguments are normally passed via command line, thus also called command line arguments in this case).

The first function, `arg_os` returns an iterator yielding these arguments as `StringOs`, while the `args` method returns iterator yielding `String`. As such, the `Args` iterator might panic if the process's argments contain invalid unicode codepoints. If this is not desired, the `args_os` method should be preferred.

Note that during the construction of `ArgsOs`, a module `sys` is used. This module is hidden from users of the standard library, as it is considered to be an implementation detail. We won't dig into it here, but interested readers can refer to the [source code of libstd](https://github.com/rust-lang/rust/tree/master/src/libstd/sys).

Let's check out the definitions of these iterator structs. Firstly the `ArgsOs`:

```ignore
pub struct ArgsOs{ inner: sys::args::Args }
```

The type implements `iterator`, `ExactSizeIterator` and `DoubleEndedIterator`, with the methods implementation being simply forwarded to its field `inner`, which also implements these traits.

Then the `Args`:

```ignore
pub struct Args { inner: ArgsOs }
```

Naturally, the type also implements `iterator`, `ExactSizeIterator` and `DoubleEndedIterator`. While most of the methods implementations are simple forwarding, as its field `inner` yields `OsString` while itself should yield `String`, a conversion is needed within the yielding methods. The conversion is done via `OsString`'s `into_string` method, which returns a `Result` with `Err` indicating conversion failure. The implementation simply `unwrap`s the result, thus would panic when encounter an argument with invalid unicode codepoints, as we stated above.

# Environment Variables

Similarly, we have two functions to retrive environment Variables:

```ignore
pub fn vars_os() -> VarsOs {
    VarsOs { inner: sys::os_imp::env() }
}

pub fn vars() -> Vars {
    Vars { inner: vars_os() }
}
```

 The corresponding struct definitions are also similar:

 ```ignore
 pub struct VarsOs { inner: sys::os_imp::Env }

 pub struct Vars { inner: VarsOs }
 ```

 These iterators yield a (variable, value) pair of strings, for all the environment variables of the current process. It is a snapshot at the time of the function invocation, respectively. As such, modification after the `vars`(`_os`) invocation to the environment variables won't be reflected in the returned iterator.

 Similarly, `VarsOs` yields `(OsString, OsString)` while `Vars` yields `(String, String)`. The later one also might panic. However, they only have the `Iterator` trait implemented.

Additionally, there are `var` methods for specific variable query:

```ignore
pub fn var_os<K>(key: K) -> Option<OsString> {
    _var_os(key.as_ref())
}

fn _var_os(key: &OsStr) -> Option<OsString> {
    sys::os_imp::getenv(key).unwrap_or_else(|e| {
        panic!("failed to get environment variable `{:?}`: {}", key, e)
    })
}

pub fn var<K>(key: K) -> Result<String, VarError>
    where K: AsRef<OsStr> {
    _var(key.as_ref())
}

fn _var(key: &OsStr) -> Result<String, VarError> {
    match var_os(key) {
        Some(s) => s.into_string().map_err(VarError::NotUnicode),
        None => Err(VarError::NotPresent)
    }
}
```

`var_os` would fetch the environment variable `key` as an `OsString`, returning `None` if the variable isn't set, while `var` fetches the corresponding variable as an `String`, returning an `VarError` when failed. The returned error message indicates why the invocation fails (either not present, orr the variable is not valid unicode).

Finally, we have `set_var` to set a specific environment variable, and `remove_var` to remove a specific variable.

```ignore
pub fn set_var<K, V>(k: K, v: V)
    where K: AsRef<OsStr>, V: AsRef<OsStr> {
    _set_var(k.as_ref(), v.as_ref())
}

fn _set_var(k: &OsStr, v: &OsStr) {
    sys::os_imp::setenv(k, v).unwrap_or_else(|e| {
        panic!("failed to set environment variable `{:?}` to `{:?}`: {}", k, v, e)
    })
}

pub fn remove_var<K>(k: K)
    where K: AsRef<OsStr> {
    _remove_var(k.as_ref())
}

fn _remove_var(k: &OsStr) {
    sys::os_imp::unsetenv(k).unwrap_or_else(|e| {
        panic!("failed to remove environment variable `{:?}`: {}", k, e)
    })
}
```

`set_var(k, v)` would set environment variable `k` to value `v` for the current process. `remove_var(k)` would remove ev `k` from the environment of the current process.

Note that, all the above four methods would panic if `key` provided is empty, or contains an ASCII `=`/`\0`; or when the value `v` contains `\0`.

Also note that these methods delegate the real work to their underscore-prefixed implementation functions. This pattern is common when dealing with generic functions. In this case, only the work of `as_ref` needs to be generic, separating such part out from the actual non-generic implementation can help reduce the size of the actual binary code generated.

# Directories

Next, we have a set of methods for some specific directorys.

`current_dir` and `set_current_dir` deal with the current working directory of the process:

```ignore
pub fn current_dir() -> io::Result<PathBuf> {
    sys::os_imp::getcwd()
}

pub fn set_current_dir<P>(p: P) -> io::Result<()>
    where P: AsRef<Path> {
    sys::os_imp::chdir(p.as_ref())
}
```

`current_dir` returns the directory as a `PathBuf`. It will returns an `Err` if the current working directory value is invalid, like when:
* Current directory does not exist.
* There are insufficient permissions to access the current directory.

`set_current_dir` sets the directory, returning whether the change was completed successfully or not.

Similarly, `current_exe` returns the full filesystem path of the current running executable file:

```ignore
pub fn current_exe() -> io::Result<PathBuf> {
    sys::os_imp::current_exe()
}
```

Based on the specific platform, the invocation can fail for many reasons. Such failure would be captured by the returned `Err` variant. Also note that, the returned path might be an intermediate symlinks, not necessarily the actual "real path".

Then we have `home_dir`, which returns the current user's home directory:

```ignore
pub fn home_dir() -> Option<PathBuf> {
    sys::os_imp::home_dir()
}
```

the value of the `HOME` environment variable would be returned if it is set and not empty. Otherwise, on Unix systems, the invocation tries to determine the home directory by invoking the `getpwuid_r` function on the UID of the current user; while on Windows systems, it will further explore the value of `USERPROFILE` environment variable, if that is also not set or empty, the invocation would call the Win32 function `GetUserProfileDirectory` to determine the path.

Finally, `temp_dir` returns the path of a temporary directory:

```ignore
pub fn temp_dir() -> PathBuf {
    sys::os_imp::temp_dir()
}
```

On Unix systems, it returns the `TMPDIR` ev if set and not empty. Otherwise, for non-Android, it returns `/tmp`; for Android, it returns `/data/local/tmp`.

On Windows systems, it returns the value of `TMP`, `TEMP` or `USERPROFILE` ev, in order, if any are set and not empty. Otherwise, it returns the path of the Windows directory. This is actually identical to the behavior of the Win32 function `GetTempPath`, which the function uses internally.

# PATH

Finally, for the `PATH` environment variable, we have two additional functions. Either on Unix or on Windows systems, this environment variable has a special meaning of specifying the universally directly callable binaries' directories. However, the separating strategies of these pathes are not identical across platforms. As such, the standard library provides the following two functuons for `PATH` ev manipulation:

```ignore
pub fn split_paths<T>(unparsed: &T) -> SplitPaths
    where T: AsRef<OsStr> + ?Sized {
    SplitPaths { inner: sys::os_imp::split_paths(unparsed.as_ref()) }
}

pub struct SplitPaths<'a> {
    inner: sys::os_imp::SplitPaths<'a>
}

impl<'a> Iterator for SplitPaths<'a> {
    type Item = PathBuf;
    fn next(&mut self) -> Option<PathBuf> {
        self.inner.next()
    }
    fn size_hint(&self) -> (usize, Option<usize>) {
        self.inner.size_hint()
    }
}
```

Function `split_paths(paths)` takes an unparsed `PATH` value `paths`, and returns an iterator over the actual paths contained in `paths`. The iterator yields these paths as `PathBuf`s.

Function `join_paths` does the contrary:

```ignore
pub fn join_paths<I, T>(paths: I) -> Result<OsString, JoinPathsError>
    where I: IntoIterator<Item=T>, T: AsRef<OsStr> {
    sys::os_imp::join_paths(paths.into_iter()).map_err(|e| {
        JoinPathsError{ inner: e}
    })
}
```

It joins a collection of `paths` for the `PATH` ev, returning an `OsString` on success. If one of the input `Path`s contains invalid chracter for contructing the value of a `PATH` variable( dobule quote on Windows, colon on Unix), the function returns an `Err`.

# Constants

The module also contains a submodule, `const`, which contains some constants associated with the current building target.

The provided infomation include:

* `ARCH: &'static str`: A string describing the architecture of the CPU that is currently in use. Possible values include "x86", "x86_64", "arm", "aarch64", "mips", "mips64", "powerpc", "powerpc64", etc.
* `FAMILIY: &'static str`: A string describing the family of the OS. Possible values include "unix", "windows".
* `OS: &'static str`: A string describing the specific OS. Possible values include "linux", "macos", "ios", "freebsd", "android", "windows".
* `DLL_PREFIX: &'static str`: A string specifying the filename prefix used for shared libraries on the platform. Possible values include "lib", ""(an empty string).
* `DLL_SUFFIX: &'static str`: A string specifying the filename suffix used for shared libraries on the platform. Possible values include ".so", ".dylib", ".dll".
* `DLL_EXTENSION: &'static str`: A string specifying the file extension used for shared libraries on the platform that goes after the last dot. Possible values include "so", "dylib", "dll".
* `EXE_SUFFIX: &'static str`: A string specifying the filename suffix used for executable binaries on the platform. Possible values include ".exe", ".nexe", ".pexe", "".
* `EXE_EXTENSION: &'static str`: A string specifying the file extension used for executable binaries on the platform. Possible values include "exe", "".
