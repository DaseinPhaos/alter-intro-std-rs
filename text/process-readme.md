% std::process

The module deals with processes.

First of all, the module defines

```ignore
pub fn exit(code: i32) -> !
```

to terminate the current process with the specified exit code. The function never returns (thus the `-> !`). The exit code would be passed to the underlying OS.

Note that upon invoking this function, no destructors on the current stack or any other thread's stack will get invoked.

---

Next, to spawn a new process, one starts with the process builder, `Command`:

```ignore
pub struct Command {/*...*/}
```

To create such a builder:

```ignore
impl Command {
    pub fn new<S: AsRef<OsStr>>(program: S) -> Command {/*...*/}
}
```

`program` specifies the path at which the process to be launched lies.

Initially, the command contains no arguments. To add an argument to pass to the program:

```ignore
impl Command {
    pub fn arg<S: AsRef<OsSttr>>(&mut self, arg: S) -> &mut Command {/*...*/}
}
```

Note that the method returns a mutable reference of `self`. This allows multiple argument additions to be chained together.


ALternatively, one can add multiple arguments to the program with a single invocation:

```ignore
impl Command {
    pub fn arg<S: AsRef<OsSttr>>(&mut self, args: &[S]) -> &mut Command {/*...*/}
}
```

Initially, the child process would inherit the current process's environment variables. To add or update the environment of the child:

```ignore
impl Command {
    pub fn env<K, V>(&mut self, key: K, val: V) -> &mut Command
        where K: AsRef<OsStr>, V: AsRef<OsStr> {/*...*/}
}
```

To remove an ev for the child:

```ignore
impl Command {
    pub fn env_remove<K: AsRef<OsStr>>(&mut self, key: K) -> &mut Command {/*...*/}
}
```

To completely clear the entire ev for the child:

```ignore
impl Command {
    pub fn env_clear(&mut self) -> &mut Command {/*...*/}
}
```

Initially, the child process would inherit parent's working directory (at the time of the creation of the `Command`). To change the working directory at the time of spawning for the child:

```ignore
impl Command [
    pub fn current_dir<P: AsRef<Path>>(&mut self, dir: P) -> &mut Command  {/*...*/}
]
```

Initially, the child process inherits parent's stdin/stdout and stderr. To configure these three:

```ignore
impl Command {
    pub fn stdin(&mut self, cfg: Stdio) -> &mut Command {/*...*/}

    pub fn stdout(&mut self, cfg: Stdio) -> &mut Command {/*...*/}

    pub fn stderr(&mut self, cfg: Stdio) -> &mut Command {/*...*/}
}
```

The configuration was enraptured by another structure `Stdio`, the definition of which is irrelevant, but can be constructed by

```ignore
impl Stdio {
    // A new pipe should be arranged to connect the parent and child.
    pub fn piped() -> Stdio {/*...*/}

    // The child inherits from the corresponding parent descriptor.
    pub fn inherit() -> Stdio {/*...*/}

    // The stream would be ignored.
    pub fn null() -> Stdio {/*...*/}
}
```

Finally, `Command` provides three ways to actually spawn a process.

To spawn a child process, wait for it to finish, and collect its exit status, one call

```ignore
impl Command {
    pub fn status(&mut self) -> Result<ExitStatus> {/*...*/}
}
```

The `ExitStatus` returned provides

```ignore
impl ExitStatus {
    pub fn success(&self) -> bool {/*...*/}

    pub fn code(&self) -> Option<i32> {/*...*/}
}
```

To spawn a child process, wait for it to finish, and collects all its output, one call

```ignore
impl Command {
    pub fn output(&mut self) -> Result<Output> {/*...*/}
}
```

The `Output` returned is defined as

```ignore
pub struct Output {
    pub status: ExitStatus,
    pub stdout: Vec<u8>,
    pub stderr: Vec<u8>,
}
```

To spawn a child process and returns a handle for it, one call

```ignore
impl Command {
    pub fn spawn(&mut self) -> Result<Child> {/*...*/}
}
```

A `Child` is a handle structure representing a running or exited process:

```ignore
pub struct Child {
    pub stdin: Option<ChildStdin>,
    pub stdout: Option<ChildStdout>,
    pub stderr: Option<ChildStderr>,
    // ...
}
```

Notice that it also exposes fields representing handles to child process's stdio handles:

- `ChildStdin` implements `std::io::Write`, so that we can write into it.
- `ChildStdout` and `ChildStderr` implements `std::io::Read`, so that we can read from it.

To force to kill the child process:

```ignore
impl Child {
    pub fn kill(&mut self) -> Result<()>  {/*...*/}
}
```

To return the OS-assigned identifier of the child process:

```ignore
impl Child {
    pub fn id(&self) -> u32 {/*...*/}
}
```

To wait for the child process to exit, either returning its `ExitStatus` or collecting all its `Output`:

```ignore
impl Child {
    pub fn wait(&mut self) -> Result<ExitStatus> {/*...*/}

    pub fn wait_with_output(self) -> Result<Output> {/*...*/}
}
```
