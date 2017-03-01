% std::io

> https://doc.rust-lang.org/std/io/

This module provides general support for IO.

# IO Traits

At the very core, the module defines some traits representing objects that are capable of doing IO.

## Read

```ignore
pub trait Read {
    fn read(&mut self, buf: &mut [u8]) -> Result<usize>;
}
```

The `Read` trait represents a `Reader`, from which one can read data. It's sole required method, `reader.read(dst)`, attempts to pull bytes from `reader` into `dst`. The returned value represents how many bytes are actually read (on success). The function might block (or not). If the returned value is `Ok(n)`, then ` 0 <= n <= buf.len()` must hold. `n == 0` means either `EOF` has been reached or the buffer specified was zero length.

The returned `Result` is actually defined as:

```ignore
type Result<T> = Result<T, Error>;
```

We'd come to this `Error` type later.

With the `read` method implemented, `Read` also provides some additional default-implemented methods, including:

```ignore
pub trait Read {
    fn read_to_end(&mut self, buf: &mut Vec<u8>) -> Result<usize> {
        read_to_end(self, buf)
    }
}
```

The method would try to read all bytes into the provided `Vec` until `EOF` is reached. All bytes read will be appended into the `buf`. Internally, it calls the module-wide implementation function `read_to_end`, which is defined as:

```ignore
const DEFAULT_BUF_SIZE: usize = ::sys_common::io::DEFAULT_BUF_SIZE;
fn read_to_end<R>(r: &mut R, buf: &mut Vec<u8>) -> Result<usize>
    where R: Read + ?Sized {
    let start_len = buf.len();
    let mut len = start_len;
    let mut new_write_size = 16;
    let ret;
    loop {
        if len == buf.len() {
            if new_write_size < DEFAULT_BUF_SIZE {
                new_write_size *= 2;
            }
            buf.resize(len + new_write_size, 0);
        }

        match r.read(&mut buf[len..]) {
            Ok(0) => {
                ret = Ok(len - start_len);
                break;
            }
            Ok(n) => len += n,
            Err(ref e) if e.kind() == ErrorKind::interrupted => {}
            Err(e) => {
                ret = Err(e);
                break;
            }
        }
    }
    buf.truncate(len);
    ret
}
```

The function uses a loop to repeatly read bytes from reader `r`, until an EOF is reached (indicated by an `Ok(0)`) or an error other than interruption is encountered. At the beginning of each loop, it checks whether the `buf`fer can read in any more bytes. If not, it allocates more capacity for the buffer. The allocation is done adaptively, such that upon each new allocation, the additional size to be allocated would double. This reduces the total number of allocations needed. After the loop, the buffer is then truncated to the actual length of bytes it currently holds, and the result `ret` is returned.

There is also a method to read the bytes into a `&mut String`:

```ignore
pub trait Read {
    fn read_to_string(&mut self, buf: &mut String) -> Result<usize> {
        append_to_string(buf, |b| read_to_end(self, b))
    }
}
```

Internally, it makes use of another implementation function:

```ignore
fn append_to_string<F>(buf: &mut String, f: F) -> Result<usize>
    where F: FnOnce(&mut Vec<u8>) -> Result<usize> {
    struct Guard<'a> { s: &'a mut Vec<u8>, len: usize, }
    impl<'a> Drop for Guard<'a> {
        fn drop(&mut self) {
            unsafe { self.s.set_len(self.len); }
        }
    }

    unsafe {
        let mut g = Guard { len: buf.len(), s: buf.as_mut_vec() };
        let ret = f(g.s);
        if str::from_utf8(&g.s[g.len..]).is_err() {
            ret.and_then(|_| {
                Err(Error::new(ErrorKind::InvalidData,
                    "stream did not contain valid UTF-8"))
            })
        } else {
            g.len = g.s.len();
            ret
        }
    }
}
```

`read_to_string` forwards the string buffer and a closure `|b| read_to_end(self, b)` to `append_to_string`. Inside `append_to_string`, the `String` is interpreted as a `&mut Vec[u8]` and was passed to the closure. After the actual reading has complete within the closure, the function invokes `str::from_utf8` to check whether the additionally read bytes form valid `utf-8`, and returns an error if not. Otherwise, the resultant buffer is returned. The method utilizes a `Guard` struct to ensure that the additionally read bytes would be properly dropped if they are not valid UTF8.

Implemented this way with `unsafe`, the function ensures that no redundant memory-allocation or UTF8 checking is needed.

Next there is a method to read for exact bytes that the provided buffer can hold:

```ignore
pub trait Read {
    fn read_exact(&mut self, mut buf: &mut [u8]) -> Result<()> {
        while !buf.is_empty() {
            match self.read(buf) {
                Ok(0) => break,
                Ok(n) => { let tmp = buf; buf = &mut tmp[n..]; }
                Err(ref e) if e.kind() == ErrorKind::interrupted => {}
                Err(e) => return Err(e),
            }
        }

        if !buf.is_empty() {
            Err(Error::new(ErrorKind::UnexpectedEof,
                "failed to fill whole buffer"))
        } else {
            Ok(())
        }
    }
}
```

The implementation is quite straightforward. It makes use of a mutable reference, `buf`. When additional bytes is read into the buffer, `buf` shrinks. If the length of `buf` shrinks to `0`, then exact bytes of the original buffer size have been read. Early end of looping caused by errors causes early returns. Early end of looping caused by `EOF` is caught by another round of checking.

Next there is a utility method to create a mutable reference of this reader:

```ignore
pub trait Read {
    fn by_ref(&mut self) -> &mut Self where Self: Sized { self }
}
```

Next there is a method to transform the reader into an `Iterator` over bytes:

```ignore
pub trait Read {
    fn bytes(self) -> Bytes<Self> where Self: Sized {
        Bytes { inner: self }
    }
}
```

The definition of the returned iterator is quite straightforward:

```ignore
pub struct Bytes<R> { inner: R, }

impl<R: Read> Iterator for Bytes<R> {
    type Item = Result<u8>;
    fn next(&mut self) -> Option<Result<u8>> {
        read_one_byte(&mut self.inner)
    }
}

// utility method
fn read_one_byte(reader: &mut Read) -> Option<Result<u8>> {
    let mut buf = [0];
    loop {
        return match reader.read(&mut buf) {
            Ok(0) => None,
            Ok(..) => Some(Ok(buf[0])),
            Err(ref e) if e.kind() == ErrorKind::interrupted => continue,
            Err(e) => Some(Err(e)),
        }
    }
}
```

Next, there is a method to consume the reader and chains it with another one:

```ignore
pub trait Read {
    fn chain<R: Read>(self, next: R) -> Chain<Self, R>
        where Self: Sized {
        Chain { first: self, second: next, done_first: false}
    }
}
```

The resultant `Chain<Self, R>` is also a reader, defined as:

```ignore
pub struct Chain<T, U> {
    first: T,
    second: U,
    done_first: bool,
}

impl<T: Read, U: Read> Read for Chain<T, U> {
    fn read(&mut self, buf: &mut [u8]) -> Result<usize> {
        if !self.done_first {
            match self.first.read(buf)? {
                0 if buf.len() !- 0 => { self.done_first = true; }
                n => return Ok(n),
            }
        }
        self.second.read(buf)
    }
}
```

It also implements `BufRead`, which would be covered later.

Finally, the trait defines a `take` method that consumes `self` and returns an adapter that will read at most `limit` bytes:

```ignore
pub trait Read {
    fn take(self, limit: u64) -> Take<Self> where Self: Sized {
        Take { inner: self, limit: limit }
    }
}
```

`Take` is defined as follows:

```ignore
pub struct Take<T> {
    inner: T,
    limit: u64,
}

impl<T> Take<T> {
    pub fn limit(&self) -> u64 { self.limit }

    pub fn into_inner(self) -> T { self.inner }
}

impl<T: Read> Read for Take<T> {
    fn read(&mut self, buf: &mut [u8]) -> Result<usize> {
        if self.limit == 0 { return Ok(0);}
        let max = cmp::min(buf.len() as u64, self.limit) as usize;
        let n = self.inner.read(&mut buf[..max])?;
        self.limit -= n as u64;
        Ok(n)
    }
}
```

Notice that the `limit` field of `Take<T>` would shrink with any successful `read`. The type also implements `BufRead`, which we'd cover later.

## Write

This trait represents a byte sink into which bytes can be written.

It requires two method to be implemented:

```ignore
pub trait Write {
    fn write(&mut self, buf: &[u8]) -> Result<usize>;
    fn flush(&mut self) -> Result<()>;
}
```

The `write` method tries to write the content of the `buf` into the writer. But the entire content of the buffer might not be all written. On success the number of bytes actually written would be returned. Calls to `write` don't make any assumption about whether the operation would block or not. If the returned value is some `Ok(n)`, then it must be guaranteed that `0 <= n <= buf.len()`. Returned value of `0` typically means that the sink is no longer able to accept bytes anymore (when `buf.len() > 0`, note that this is not considered as an `Err`).

The `flush` method tries to flush the output stream, ensuring that all the intermediately buffered contents reaches their true destination.

additionally, the trait provides

```ignore
pub trait Write {
    fn write_all(&mut self, mut buf: &[u8]) -> Result<()> {
        while !buf.is_empty() {
            match self.write(buf) {
                Ok(0) => return Err(Error::new(ErrorKind::WriteZero, "failed to write whole buffer")),
                Ok(n) => buf = &buf[n..],
                Err(ref e) if e.kind() == ErrorKind::interrupted => {}
                Err(e) => return Err(e),
            }
        }
        Ok(())
    }
}
```

The method tries to write all the contents in `buf` into the sink.

Additional, there is:

```ignore
pub trait Write {
    fn write_fmt(&mut self, fmt: fmt::Arguments) -> Result<()> {
        struct Adaptor<'a, T: ?Sized + 'a> {
            inner: &'a mut T,
            error: Result<()>,
        }

        impl<'a, T: Write + ?Sized> fmt::Write for Adaptor<'a, T> {
            fn write_str(&mut self, s: &str) -> fmt::Result {
                match self.inner.write_all(s.as_bytes()) {
                    Ok(()) => Ok(()),
                    Err(e) => {
                        self.error = Err(e);
                        Err(fmt::Error)
                    }
                }
            }
        }

        let mut output = Adaptor { inner: self, error: Ok(()) };
        match fmt::write(&mut output, fmt) {
            Ok(()) => Ok(()),
            Err(..) => {
                if output.error.is_err() {
                    output.error
                } else {
                    Err(Error::new(ErrorKind::Other, "formatter error"))
                }
            }
        }
    }
}
```

This method is primarily used to incorporate with the formatting macros. It is rarely called alone explicitly. To write a formatted string into the sink, the `write!` macro should be preferred.

Finally, there is:

```ignore
pub trait Write {
    fn by_ref(&mut self) -> &mut Self where Self: Sized { self }
}
```

to get a mutable reference.

## Seek

```ignore
pub trait Seek {
    fn seek(&mut self, pos: SeekFrom) -> Result<u64>;
}
```

The `Seek` trait represents an IO object with a "cursor", which represents the current position in the IO stream represented by the object. The required method `seek` attempts to move that cursor. Such moving should be relative, thus the possible relative positions (along with how many bytes would be moved) are passed in as the `SeekFrom` enum, which is defined as:

```ignore
#[derive(Copy, PartialEq, Eq, Clone, Debug)]
pub enum SeekFrom {
    Start(u64),
    End(i64),
    Current(i64),
}
```

The `Start` variant means an offset from the "start" of the stream. It contains a `u64`, thus, only seeking *after* the starting point is valid. The `End` variant means an offset from the "end" of the stream, containing an `i64`, it is considered valid to seek before and beyond the ending point of the stream. The `Current` variant means an offset from the "current" position in the stream.

The `seek` method takes such an offset, and tries to move the cursor to the specified position. If the invocation succeed, the returned `Result` would be the distance from the starting point to the new position.

## BufRead

```ignore
pub trait BufRead: Read {
    fn fill_buf(&mut self) -> Result<&[u8]>;
    fn consume(&mut self, amt: usize);
}
```

`BufRead` represents a specific type of `Read`er that has an internal buffer. Having a buffer allows extra ways of reading to become efficient.

The trait requires two methods to be implemented.

`fill_buf` should fill the internal buffer of the object, returning the buffer contents upon success. `buf.consume(amt)` tells the buffer that `amt` bytes have been consumed from the internal buffer, thus should no longer be returned be succeeding `read`ings. These two functions are relatively low-level and should be used in pair.

With these two methods implemented, the trait provides some high-level utility methods for daily use:

```ignore
pub trait BufRead: Read {
    fn read_until(&mut self, byte: u8, buf: &mut Vec<u8>) -> Result<usize> {
        read_until(self, byte, buf)
    }
}
```

The method tries to read all bytes into `buf` until the delimiter `byte` or EOF is reached. The delimiter (if found) will also be appended to the target `byf`, Upon success, the function will return the total bytes read. Internally, it forwards the call to the implementation function, which is defined as:

```ignore
fn read_until<R>(r: &mut R, delim: u8, buf: &mut Vec<u8>) -> Result<usize> {
    let mut read = 0;
    loop {
        let (done, used) = {
            let available = match r.fill_buf() {
                Ok(n) = n,
                Err(ref e) if e.kind == ErrorKind::interrupted => continue,
                Err(e) => return Err(e)
            };

            match memchr::memchr(delim, available) {
                Some(i) => {
                    buf.extend_from_slice(&available[..i+1]);
                    (true, i+1)
                }
                None => {
                    buf.extend_from_slice(available);
                    (false, available.len())
                }
            }
        };
        r.consume(used);
        read += used;
        if done || used == 0 {
            return Ok(read)
        }
    }
}
```

The function also demonstrates the usage of `fill_buf` and `consume`. The function calls `r.fill_buf` inside a loop to fill the internal buffer of `r` and returns a reference of it (`available`) if the operation succeed. It then uses the extremely fast `memchr::memchr` to find the delimiter `delim` in `available`. If found, the bytes from the start to the index of delimiter in the internal buffer would be consumed and appended into the target buffer. If not, all the bytes in the internal buffer would be consumed into the target buffer. Depending on whether the delimiter has been found or EOF has been reached (indicated by 0 bytes available in the internal buffer), the function would jump out of the loop and return the total bytes read.

Additionally, the trait provides:

```ignore
pub trait BufRead {
    fn read_line(&mut self, buf: &mut String) -> Result<usize> {
        append_to_string(buf, |b| read_until(self, b'\n', b))
    }
}
```

which would attempt to read all bytes until a newline character (`0xa` byte) is reached. The bytes read would be appended to the provided `String` buffer if they form valid UTF8.

Internally, it makes use of the `append_to_string` function, which was introduced above. Now we've uncovered the reason of this function's signature design: reduce code duplication.

The trait also provides

```ignore
pub trait BufRead {
    fn split(self, byte: u8) -> Split<Self> where Self: Sized {
        Split { buf: self, delim: byte }
    }
}
```

which would consume this reader and returns an iterator over the contents of it, split on the `byte` given.

The iterator is defined as:

```ignore
pub struct Split<B> {
    buf: B,
    delim: u8,
}

impl<B: BufRead> Iterator for Split<B> {
    type Item = Result<Vec<u8>>;

    fn next(&mut self) -> Option<Result<Vec<u8>>> {
        let mut buf = Vec::new();
        match self.buf.read_until(self.delim, &mut buf) {
            Ok(0) => None, // the underlying reader has reached EOF, so return `None`
            Ok(_n) => {
                if buf[buf.len() - 1] == self.delim {
                    buf.pop(); // the last byte is the delimiter, so pop it out
                }
                Some(Ok(buf))
            }
            Err(e) => Some(Err(e))
        }
    }
}
```

Note that the iterator yields a `Result<Vec<u8>>`, meaning that the last `Err` encountered will be propagated and returned to the caller of the trait.

Finally, the trait provides

```ignore
pub trait BufRead {
    fn lines(self) -> Lines<Self> where Self: Sized {
        Lines{ buf: self }
    }
}
```

which consumes the reader and returns an iterator over the lines of it content.

The definition of the returned struct is similar to `Split`:

```ignore
pub struct Lines<B> {
    buf: B,
}

impl<B: BufRead> Iterator for Lines<B> {
    type Item = Result<String>;

    fn next(&mut self) -> Option<Result<String>> {
        let mut buf = String::new();
        match self.buf.read_line(&mut buf) {
            Ok(0) => None,
            Ok(_n) => {
                if buf.ends_with("\n") {
                    buf.pop();
                    if buf.ends_with("\r") {
                        buf.pop();
                    }
                }
                Some(Ok(buf))
            }
            Err(e) => Some(Err(e))
        }
    }
}
```

Note that it uses an additional `if` to deal with `/r`.

# Buffered IO

IO often requires a lot of interaction with some sort of "external device", which might be really inefficient. As such, buffering is commonly used to help reduce the overhead. To capture this pattern, besides the `BufRead` trait, the module also provides three wrapper structs.

## BufReader

First comes the `BufReader`:

```ignore
pub struct BufReader<R> {
    inner: R,
    buf: Box<[u8]>,
    pos: usize, // current position into the buffer
    cap: usize, // total buffered bytes in the buffer
}
```

It is meant to be a buffered wrapper around another reader of type `R`. To construct it, the module exposes:

```ignore
impl<R: Read> BufReader<T> {
    pub fn new(inner: R) -> BufReader<R> {
        BufReader::with_capacity(DEFAULT_BUF_SIZE, inner)
    }

    pub fn with_capacity(cap: usize, inner: R) -> BufReader<R> {
        BufReader {
            inner: inner,
            buf: vec![0; cap].into_boxed_slice(),
            pos: 0,
            cap: 0,
        }
    }
}
```

Note that once constructed the capacity of the internal buffer can't be changed.

As expected, the struct implements `BufRead`:

```ignore
impl<R: Read> BufRead for BufReader<R> {
    fn fill_buf(&mut self) -> io::Result<&[u8]> {
        if self.pos >= self.cap { // buffered bytes all consumed
            debug_assert_eq!(self.pos, self.cap);

            // fetch more from the inner buffer, propagate any error
            self.cap = self.inner.read(&mut self.buf)?;

            // reset the current `pos`
            self.pos = 0;
        }

        // return the bytes available
        Ok(&self.buf[self.pos..self.cap])
    }

    fn consume(&mut self, amt: usize) {
        // consume `amt` bytes. make sure that `pos` do not go beyond `cap`.
        self.pos = cmp::min(self.pos + amt, self.cap);
    }
}

impl<R: Read> Read for BufReader<R> {
    fn read(&mut self, buf: &mut [u8]) -> io::Result<usize> {
        if self.pos == self.cap && buf.len() >= self.buf.len() {
            // We don't have any bytes buffered internally,
            // and the provided buffer's capacity is greater than
            // our internal buffer's size, so buffering is of no use,
            // we thus bypass our internal buffer
            return self.inner.read(buf);
        }

         // else, let's fetch some data from the internal buffer
        let nread = {
            let mut rem = self.fill_buf()?;
            rem.read(buf)? // note that `[u8]` also implements `Read`, so this invocation is possible
        };
        self.consume(nread); // then we `consume` these bytes
        Ok(nread) // and return the bytes consumed
    }
}
```

Additionally, for seekable inner reader, the wrapper also implements `Seek`:

```ignore
impl<R: Seek> Seek for BufReader<R> {
    fn seek(&mut self, pos: SeekFrom) -> io::Result<u64> {
        let result: u64;
        if let SeekFrom::Current(n) = pos {
            let remainder = (self.cap - self.pos) as i64;
            
            // use `checked_sub` to deal with possible underflow
            if let Some(offset) = n.checked_sub(remainder) {
                result = self.inner.seek(SeekFrom::Current(offset))?;
            } else {
                // can't seek by the actual offset, so we seek twice
                self.inner.seek(SeekFrom::Current(-remainder))?;
                self.pos = self.cap; // effectively, this empties the inner buffer
                                     // this is needed so that if any `Err` occurs
                                     // and the function is returned in the next
                                     // statement, the whole inner buffer would be
                                     // left in a valid state
                result = self.inner.seek(SeekFrom::Current(n))?;
            }
        } else {
            // Seeking from `Start` or `End` doesn't care about the current pos
            result = self.inner.seek(pos)?;
        }
        self.pos = self.cap; // empty the inner buffer
        Ok(result)
    }
}
```

The wrapper also provides some methods to convert to the underlying reader. Note that, during these conversions, leftover bytes in the wrapper's internal buffer is ignored. Thus they should be used with caution.

```ignore
impl<R: Read> BufReader<R> {
    pub fn get_ref(&self) -> &R { &self.inner }
    pub fn get_mut(&mut self) -> &mut R { &mut self.inner }
    pub fn into_inner(self) -> R { self.inner }
}
```

Additionally, the wrapper implements `Debug` for `R: Debug`.

## BufWriter

Analog to `BufReader`, `BufWriter` maintains an in-memory buffer for a wrapped writer.

```ignore
pub struct BufWriter<W: Write> {
    inner: Option<W>,
    buf: Vec<u8>,
    panicked: bool,
}
```

To construct it:

```ignore
impl<W: Write> BufWriter<W> {
    pub fn new(inner: W) -> BufWriter<W> {
        BufWriter::with_capacity(DEFAULT_BUF_SIZE, inner)
    }

    pub fn with_capacity(cap: usize, inner: W) -> BufWriter<W> {
        BufWriter {
            inner: Some(inner),
            buf: Vec::with_capacity(cap),
            panicked: false,
        }
    }
}
```

Note that once created, the capacity of the internal buffer can't be changed.

As expected, the wrapper implements `Write`:

```ignore
impl<W: Write> Write for BufWriter<W> {
    fn write(&mut self, buf: &[u8]) -> io::Result<usize> {
        if self.buf.len() + buf.len() > self.buf.capacity() {
            // currently, the internal buffer can't hold all the bytes
            // provided by the invocation, so flush the internal buffer
            self.flush_buf()?;
        }
        if buf.len() >= self.buf.capacity() {
            // if the provided bytes's length is greater than the capacity
            // of the internal buffer itself, try to directly deal with the
            // inner buffer
            self.panicked = true;
            let r = self.inner.as_mut().unwrap().write(buf);
            self.panicked = false;
            r
        } else {
            // the provided bytes can be written into the internal buffer,
            // so write it in. This is possible as `Vec<u8>` implements `write`.
            Write::write(&mut self.buf, buf)
        }
    }

    fn flush(&mut self) -> io::Result<()> {
        // flush the intenal buffer into the underlying buffer,
        // then flush the underlying buffer
        self.flush_buf().and_then(|()| self.get_mut().flush())
    }
}
```

For `W: Seek` it also implements `Seek`:

```ignore
impl<W: Write + Seek> Seek for BufWriter<W> {
    /// Seek to the offset, in bytes, in the underlying writer.
    ///
    /// Seeking always writes out the internal buffer before seeking.
    fn seek(&mut self, pos: SeekFrom) -> io::Result<u64> {
        self.flush_buf().and_then(|_| self.get_mut().seek(pos))
    }
}
```

The above implementations makes use of a utility function

```ignore
impl<W: Write> BufWriter<W> {
    fn flush_buf(&mut self) -> io::Result<()> {
        let mut written = 0;
        let len = self.buf.len();
        let mut ret = Ok(());
        while written < len {
            self.panicked = true;
            let r = self.inner.as_mut().unwrap().write(&self.buf[written...]);
            self.panicked = false;

            match r {
                Ok(0) => {
                    ret = Err(Error::new(ErrorKind::WriteZero, 
                        "failed to write the buffered data"));
                    break;
                }
                Ok(n) => written += n,
                Err(ref e) if e.kind() == io::ErrorKind::interrupted => {}
                Err(e) => { ret = Err(e); break; }
            }
        }
        if written > 0 {
            self.buf.drain(..written); // drain the written bytes
        }
        ret
    }
}
```

which attempts to flush the internal buffer's content into the inner writer. Any error encountered would be returned.

To make sure that the buffered content would be written into the underlying writer when the wrapper goes out of scope, it also implements `Drop`:

```ignore
impl<W: Write> Drop for BufWriter<W> {
    fn drop(&mut self) {
        if self.inner.is_some() && !self.panicked {
            let _r = self.flush_buf(); // dropping shouldn't panic, so the
                                       // returned error (if any) is ignored
        }
    }
}
```

Note that this is also where the `panicked` flag gets used: when `true` the flag indicates that the `drop`ing is because of unwinding which is caused by a panic happened during some `writ`ing invocation. When this is the case, no further writing should be performed by the `drop`ing (to prevent double-panic).

It also expose methods for conversions to the underlying writer:

```ignore
impl<W: Write> BufWriter<W> {
    pub fn get_ref(&self) -> &W { self.inner.as_ref().unwrap() }
    pub fn get_mut(&mut self) -> &mut W { self.inner.as_mut().unwrap() }
    pub fn into_inner(mut self) -> Result<W, IntoInnerError<BufWriter<W>>> {
        match self.flush_buf() {
            Err(e) => Err(IntoInnerError(self, e)),
            Ok(()) => Ok(self.inner.take().unwrap())
        }
    }
}
```

Note that unlike `BufReader`, the `into_inner` method of `BufWriter` actually *does* try to flush the buffered content into `inner`. If the flushing fails, the value of `BufWriter` as well as the failing `Error` would be returned inside an `IntoInnerError`. Also note that this method is the reason why `BufWriter<T>` wraps around an `Option<T>` (rather than simply wrapping `T`). Unlike `BufReader<T>`, `BufWriter` implements `Drop`. As such, it is not allowed to directly move the value of its field out.[^1] Thus another layer of wrapping.

[^1]: To get a further explanation check out [The Rustonomicon](https://doc.rust-lang.org/nomicon/destructors.html).

Also note that the wrapper implements `Debug` if the underlying writer implements `Debug`.

## LineWriter

The module also provides a `LineWriter`, which wraps a writer and buffers output to it, flushing whenever a newline character (`b'\n'`) is encountered.

```ignore
pub struct LineWriter<W: Write> {
    inner: BufWriter<W>,
}
```

To construct it:

```ignore
impl<W: Write> LineWriter<W> {
    pub fn new(inner: W) -> LineWriter<W> {
        LineWriter::with_capacity(1024, inner)
    }

    pub fn with_capacity(cap: usize, inner: W) -> LineWriter<W> {
        LineWriter { inner: BufWriter::with_capacity(cap, inner) }
    }
}
```

It implements `Write` as we'd expect:

```ignore
impl<W: Write> Write for LineWriter<W> {
    fn write(&mut self, buf: &[u8]) -> io::Result<usize> {
        match memchr::memchr(b'\n', buf) {
            Some(i) => {
                let n = self.inner.write(&buf[..i+1])?;
                if n != i + 1 || self.inner.flush().is_err() {
                    return Ok(n); // don't return errors on partial writes. this is kind of fucked up
                }
                self.inner.write(&buf[i+1..]).map(|i| n + i)
            }
            None => self.inner.write(buf),
        }
    }

    fn flush(&mut self) -> io::Result<()> {
        self.inner.flush()
    }
}
```

It also implements `Debug` if the underlying writer implements it.

# Special Adapters

The module also provides some special reader/writers.

## Empty

```ignore
pub struct Empty { _priv: () }
```

The struct implements `Read` and `BufRead` as follows:

```ignore
impl Read for Empty {
    fn read(&mut self, _buf: &mut [u8]) -> io::Result<usize> { Ok(0) }
}

impl BufRead for Empty {
    fn fill_buf(&mut self) -> io::Result<&[u8]> { Ok(&[]) }
    fn consume(&mut self, _n: usize) {}
}
```

Thus, it represents an "empty" reader which will always return EOF.

To construct it, one calls the module level function:

```ignore
pub fn empty() -> Empty { Empty { _priv: () } }
```

## Repeat

```ignore
pub struct Repeat { byte: u8 }
```

The struct implements `Read` as:

```ignore
impl Read for Repeat {
    fn read(&mut self, buf: &mut [u8]) -> io::Result<usize> {
        for slot in &mut *buf {
            *slot = self.byte;
        }
        Ok(buf.len())
    }
}
```

Thus, it represents a reader which yields one single `byte` over and over again.

To construct it, one calls the module level function:

```ignore
pub fn repreat(byte: u8) -> Repeat { Repeat { byte: byte } }
```

## Sink

```ignore
pub struct Sink { _priv: () }
```

The struct implements `Write` as:

```ignore
impl Write for Sink {
    fn write(&mut self, buf: &[u8]) -> io::Result<usize> { Ok(buf.len()) }
    fn flush(&mut self) -> io::Resut<()> { Ok(()) }
}
```

Thus, it represents a "sink" that can take any input and throw them into the void.

To construct it, one call the module level function

```ignore
pub fn sink() -> Sink { Sink { _priv: () } }
```

## Cursor

```ignore
pub struct Cursor<T> {
    inner: T,
    pos: u64,
}
```

The struct is meant to be a wrapper around some other datastream `T`, providing a "cursor" pointing into it.

Construction is straightforward:

```ignore 
impl<T> Cursor<T> {
    pub fn new(inner: T) -> Cursor<T> {
        Cursor { pos: 0, inner: inner }
    }
}
```

For `T: AsRef[u8]`, which can be considered as a readable datastream, the module provides implementation for `BufRead` and `Seek`:

```ignore
impl<T> BufRead for Cursor<T> where T: AsRef<[u8]> {
    fn fill_bf(&mut self) -> io::Result<&[u8]> {
        // `amt` is the index of the next byte available
        // the comparison is neccessary, as we don't guard against
        // overflows when incrementing `self.pos`.
        let amt = cmp::min(self.pos, self.inner.as_ref().len() as u64);
        Ok(&self.inner.as_ref()[(amt as usize)..])
    }

    fn consume(&mut self, amt: usize) { self.pos += amt as u64; }
}

impl<T> Read for Cursor<T> where T: AsRef<[u8]> {
    fn read(&mut self, buf: &mut [u8]) -> io::Result<usize> {
        let n = Read::read(&mut self.fill_buf()?, buf)?;
        self.pos += n as u64;
        Ok(n)
    }
}

impl<T> Seek for Cursor<T> where T: AsRef<[u8]> {
    fn seek(&mut self, style: SeekFrom) -> io::Result<u64> {
        let pos = match style {
            SeekFrom::Start(n) => { self.pos = n; return Ok(n); } // early return here, caution!
            SeekFrom::End(n) => self.inner.as_ref().len() as i64 + n;
            SeekFrom::Current(n) => self.pos as i64 + n,
        };

        if pos < 0 {
            Err(Error::new(ErrorKind::InvalidInput,
                "invalid seek to a negative position"));
        } else {
            self.pos = pos as u64;
            Ok(self.pos)
        }
    }
}
```

For `T: &'a mut [u8]`, `T: Vec<u8>`, `T: Box<[u8]>` which can all be considered as writable datastream, the module also implements `Write`:

```ignore
impl<'a> Write for Cursor<&'a mut [u8]> {
    #[inline]
    fn write(&mut self, data: &[u8]) -> io::Result<usize> {
        let pos = cmp::min(self.pos, self.inner.len() as u64);
        let amt = (&mut self.inner[(pos as usize)..]).write(data)?;
        self.pos += amt as u64;
        Ok(amt)
    }

    fn flush(&mut self) -> io::Result<()> { Ok(()) }
}
```

The implementation for `Cursor<Vec<u8>>` and `Cursor<Box<[u8]>>` are somewhat similar, thus omitted.[^2]

[^2]: Actually, no. The implementation for `Cursor<Vec<u8>>` is a bit more complicated, as it tries to resize the vector to fit in all the bytes provided.

# Standard IO

The module also provides handle structures for standard IO streams of the process.

## Stdin

```ignore
pub struct Stdin { /* fields omitted */ }
```

This struct represents a handle to the standard input stream of the process. The handle can be considered as a shared reference to the actual underlying input buffer of the process. Accessing the underlying buffer through this handle is synchronized via a mutex.

To provide such a handle the module exposes a function

```ignore
pub fn stdin() -> Stdin { /*...*/ }
```

The handle implements `Read` as we'd expect. It also implements

```ignore
impl Stdin {
    pub fn read_line(&self, buf: &mut String) -> Result<usize> {/*...*/}
}
```

for convenient line-based reading.

### StdinLock

Actually, all the reading methods of `Stdin` is implemented via another struct,

```ignore
pub struct StdinLock<'a> {/* fields omitted */ }
```

The structure represents a locked reference to the standard input buffer. The lock is released when this lock goes out of scope.

To acquire such a lock one invokes

```ignore
impl Stdin {
    pub fn lock(&self) -> StdinLock {/*...*/}
}
```

As the locked reference can have exclusive reading access to the underlying resource, it also implements `BufRead`.

## Stdout

On the other side of the spectrum lies

```ignore
pub struct Stdout { /* fields omitted */ }
```

It is a synchronized handle to the standard output stream of the current process.

Likewise, to provide such a handle the module has

```ignore
pub fn stdout() -> Stdout {/*...*/}
```

To gain exclusive access to the underlying stream

```ignore
impl Stdout {
    pub fn lock(&self) -> StdoutLock {/*...*/}
}
```

The `Write` trait is implemented for this struct as well as its locking counterpart `StdoutLock`, as we'd expect.

# Leftovers

In addition, the module also provides

- `copy`,
    ```ignore
    pub fn copy<R, W>(reader: &mut R, writer: &mut W) -> Result<u64>
        where R: Read + ?Sized,
              W: Write + ?Sized {
        let mut buf = [0; DEFAULT_BUF_SIZE];
        let mut written = 0;
        loop {
            let len = match reader.read(&mut buf) {
                Ok(0) => return Ok(written),
                Ok(len) => len,
                Err(ref e) if e.kind() == ErrorKind::Interrupted => continue,
                Err(e) => return Err(e),
            };
            writer.write_all(&buf[..len])?;
            written += len as u64;
        }
    }
    ```
    The function tries to read the entire contents of a reader and writes it into another reader.

- `prelude` submodule
    `std::io` has a bunch of stuff defined. To alleviate importing many common IO traits, this submodule imports the core traits of `io`. Conceptually, it is defined as
    ```ignore
    pub mod prelude {
        pub use super::Read;
        pub use super::Write;
        pub use super::BufRead;
        pub use super::Seek;
    }
    ```