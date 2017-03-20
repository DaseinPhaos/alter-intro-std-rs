% std::fs

> https://doc.rust-lang.org/std/fs/

The module contains some functions and structures to manipulate the local filesystems. Extra platform-specific functionalities can be found in the extension modules `std::os::$platform_name`.

# Functions

## For Directory

First of all, to create a directory:

```ignore
pub fn create_dir<P>(path: P) -> Result<()> where P: AsRef<Path>
```

As its name suggests, the function creates a new empty directory at the provided `AsRef<std::path::Path>`. When user lacks permissions or `path` already exists, the function would return an `Err` indicating creation failure.

The module also contains

```ignore
pub fn create_dir_all<P>(path: P) -> Result<()> where P: AsRef<Path>
```

which would recursively create a directory and all its parent components if they don't already exist.

Then, to remove a directory:

```ignore
pub fn remove_dir<P>(path: P) -> Result<()> where P: AsRef<Path>
```

This function would fail if the user lacks permissions or the provided directory isn't empty. While the following

```ignore
pub fn remove_dir_all<P>(path: P) -> Result<()> where P: AsRef<Path>
```

would remove the directory and *all* its contents.

Next, to get content-infos of a directory:

```ignore
pub fn read_dir<P>(path: P) -> Result<ReadDir>
```

The function returns an iterator(`ReadDir`) over the entries within the given directory. When the provided directory doesn't exist, or the user lacks permissions, the invocation would fail.

The returned `ReadDir` implements `Iterator<Item=Result<DirEntry>`, where a `DirEntry` returns an entry(either a file, a link, or another directory). It implements the following methods:

```ignore
impl DirEntry {
    // returns the full path to the entry
    pub fn path(&self) -> PathBuf{ /*...*/ }

    // returns the metadata of the entry
    pub fn metaData(&self) -> Result<Metadata>{ /*...*/ }

    // returns the type of the entry
    pub fn file_type(&self) -> Result<FileType>{ /*...*/ }

    // returns the file name of the entry without other leading path components
    pub fn file_name(&self) -> OsString { /*...*/ }
}
```

Note the `metadata` method. Upon success it returns a `Metadata`, which is a struct containing meta-information about the entry.

Also note the `file_type` method, which returns a `FileType` upon success.

We will come to these structures later. For now let's focus on other functions defined in the module.

## For File

Now to copy a file:

```ignore
pub fn copy<P, Q>(from: P, to: Q) -> Result<u64>
    where P: AsRef<Path>, Q: AsRef<Path>
```

The function copies contents of `from` to `to`. It will *overwrite* the contents of `to`. On success, the number of bytes copied is returned. When `from` is not a file or doesn't exist, or the user lacks permissions, this invocation would fail.

Next, to remove a file:

```ignore
pub fn remove_file<P>(path: P) -> Result<()> where P: AsRef<Path>
```

The function tries to remove a file from the file system. However, depending on the platform, there is no guarantee that the file is immediately deleted. The function will return an error if `path` is not a valid file or the user lacks permissions.

## For Link

Next, to create a new hard link:

```ignore
pub fn hard_link<P, Q>(src: P, dst: Q) -> Result<()>
    where P: AsRef<Path>, Q: AsRef<Path>
```

Upon success, the `dst` will be a link pointing to `src`. The function will return an `Err` is `src` is not a file or deosn't exist.

To read a link and returns the entry that the link points to:

```ignore
pub fn read_link<P>(path: P) -> Result<PathBuf> where P: AsRef<Path>
```

## Common Manipulations

Next, to rename a file or a directory:

```ignore
pub fn rename<P, Q>(from: P, to: Q) -> Result<()>
    where P: AsRef<Path>, Q: AsRef<Path>
```

This function will return an `Err` if `from` doesn't exist or the user lacks permissions or `from` and `to` are on separate filesystems.

To query metadata about the entry:

```ignore
pub fn metadata<P>(path: P) -> Result<Metadata> where P: AsRef<Path>
```

The function will traverse symbolic links to query information about the entry. However, the next function won't:

```ignore
pub fn symlink_metadata<P>(path: P) -> Result<Metadata> where P: AsRef<Path>
```

Next, to set permissions on the entry:

```ignore
pub fn set_permissions<P>(path: P, perm: Permissions) -> Result<()> where P: AsRef<Path>
```

This function would fail if `path` doesn't exist or the user lacks permissions to change the file.

Lastly, there is a function to canonicalize a path, the returned path will have all its intermediate components normalized, and symbolic links resolved:

```ignore
pub fn canonicalize<P>(path: P) -> Result<PathBuf> where P: AsRef<Path>
```

## Metadata

Now let's take a look at `Metadata`. It implements:

```ignore
impl Metadata {
    // returns the type of the represented entry
    pub fn file_type(&self) -> FileType{ /*...*/ }

    // returns whether the represented entry is a directory
    pub fn is_dir(&self) -> bool{ /*...*/ }

    // returns whether the represented entry is a file
    pub fn is_file(&self) -> bool{ /*...*/ }

    // returns the size of the represented entry in bytes
    pub fn lent(&self) -> u64{ /*...*/ }

    // returns the permissions of the represented entry
    pub fn permissions(&self) -> Permissions{ /*...*/ }

    // returns the last modification time of the represented entry
    pub fn modified(&self) -> Result<SystemTime>{ /*...*/ }

    // returns the last access time
    pub fn accessed(&self) -> Result<SystemTime>{ /*...*/ }

    // returns the creation time
    pub fn created(&self) -> Result<SystemTime>{ /*...*/ }
}
```

It also implements `Clone`.

## FileType

The structure represents types of a filesystem entry. It implements:

```ignore
impl FileType {
    pub fn is_dir(&self) -> bool{ /*...*/ }

    pub fn is_file(&self) -> bool{ /*...*/ }

    pub fn is_symlink(&self) -> bool{ /*...*/ }
}
```

It also implements `Copy`, `Clone`, `PartialEq`, `Eq`, `Hash` and `Debug`.

## Permissions

The structure represents various permissions on a filesystem entry. Currently, it only provides one bit of info, `readonly`:

```ignore
impl Permissions {
    // returns whether the permission describes a readonly entry.
    pub fn readonly(&self) -> bool{ /*...*/ }

    // set the readonly flag for the permission.
    pub fn set_readonly(&mut selt, readonly: bool){ /*...*/ }
}
```

It also implements `Clone`, `PartialEq`, `Eq` and `Debug`.

# Builders

Addtionally, the module also exposes two builder-structs for a more object-oriented style directory creation and file opening.

## DirBuilder

The struct is a builder for directory creation. For me it has little usage, check the official doc.

## OpenOptions

The struct is a builder for file opening. It contains options to configure how a file should be opend.

Firstly, to create a new set of options:

```ignore
impl OpenOptions {
    pub fn new() -> OpenOptions{ /*...*/ }
}
```

All options are initially set to `false`.

Next, to set various options:

```ignore
impl OpenOptions {
    // Sets read access. When true, the file should be readable when opened.
    pub fn read(&mut self, read: bool) -> &mut OpenOptions{ /*...*/ }

    // Sets write access. When true, the file should be writable when opened.
    pub fn write(&mut self, write: bool) -> &mut OpenOptions{ /*...*/ }

    // Sets append acess. When true, writing will append to the file, rather
    // than overwrting from the file's start. When set, it also implicitly
    // sets the `write` flag.
    pub fn append(&mut self, append: bool) -> &mut OpenOptions{ /*...*/ }

    // Sets the truncating option. When true, the file would be truncated to
    // zero-length if it already exists. Write access should also be set for
    // the truncation to work.
    pub fn truncate(&mut self, truncate: bool) -> &mut OpenOptions{ /*...*/ }

    // Sets the creation option. When true, a new file will be created if the
    // file doesn't exist when opened. In order for this to work, `write`
    // access should be set.
    pub fn create(&mut self, create: bool) -> &mut OpenOptions{ /*...*/ }

    // Sets the creation option. When true, a new file will *always* be created
    // upon opening. In order for this to work, `write` access should be set.
    pub fn create_new(&mut self, create_new: bool) -> &mut OpenOptions{ /*...*/ }
}
```

Lastly, to open a file with current set of options:

```ignore
fn open<P>(&self, path: P) -> Result<File>{ /*...*/ }
```

The function will return the opened `File` upon success.

The builder also implements `Clone`.

### File

This is the struct returned by an `OpenOption`'s `open` method. It represents a reference to an opened file on the filesystem.

Sometimes it can be a bit verbose to always create a `OpenOption` for some simple file opening operations. As such, `File` provides two associated functions:

```ignore
impl File {
    pub fn open<P>(path: P) -> Result<File> where P: AsRef<Path>{ /*...*/ }

    pub fn create<P>(path: P) -> Result<File> where P: AsRef<Path>{ /*...*/ }
```

`File::open(p)` would try to open a file as in `OpenOptions::new().read(true).open(p)`.

`File::create(p)` would try to open a file as in `OpenOptions::new().write(true).create_new(true).open(p)`.

For metadata query we have:

```ignore
impl File {
    pub fn metadata(&self) -> Result<Metadata>{ /*...*/ }
}
```

For file length Manipulation we have

```ignore
impl File {
    pub fn set_len(&self, size: u64) -> Result<()>{ /*...*/ }
}
```

which would try to truncate or extends the underlying file to `size` bytes. All intermediate data filled would be set to `0`.

To acquire another independent reference(i.e. handle) to the underlying file:

```ignore
impl File {
    pub fn try_clone(&self) -> Result<File>{ /*...*/ }
}
```

The returned handle would be an independent handle, with all the states identical to the current one.

The underlying file resides on a disk or some other external devices. As such, synchronization should be done between the handle and the underlying file on the disk. For this we have:

```ignore
impl File {
    pub fn sync_all(&self) -> Result<()>{ /*...*/ }
}
```

to sync all OS-internal metadata to the disk before its returning.

We also have

```ignore
impl File {
    pub fn sync_data(&self) -> Result<()>{ /*...*/ }
}
```

to just synchronize file data to the filesystem. The method can be ued when content-synchronization is required, but metadata on disk is not. This requires less disk operations(which is *really* slow) than the previous method.

To set permissions on the file we have:[^unstable]

```ignore
impl File {
    pub fn set_permissions(&self, perm: Permissions) -> Result<()>{ /*...*/ }
}
```

which will try to change the permissions set on the file atomically.

[^unstable]: As of 1.15 the function is marked as unstable. However, according to the associated tracking issue, it would be stabilized with the release of 1.16.

To actually read or write to the file, `std::io::Read` and `std::io::Write` as well as `std::io::Seek` is implemented.

*Note*: The above three traits, `Read`, `Write` and `Seek` are also implemented for `&'a File`. Why bother? Because `Read`ing or `Writing` or `Seek`ing all requires a mutable reference (as they logically should). However, libstd decides that the mutability of a `File`, which is simply just a wrapper for some os handle, is irrelevant to the mutability of its pointed file's content. The handle's immutability only means that its pointing behavior won't change, nothing more. As reading/writing/seeking here deals with the pointed file's content rather than the file handle itself, it should also be usable through an immutable reference of the handle. As such, we have things like `impl<'a> Read for &'a File`. This pattern is fairly common in the standard library.