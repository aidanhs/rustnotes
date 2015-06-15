# Rust glibc static linking

By default, all Rust programs on Linux will link to the system libc along with
a number of other libraries Let's look at an example on a 64-bit linux machine
with GCC and glibc (by far the most common libc on Linux):

``` text
$ cat example.rs
fn main() {}
$ rustc example.rs
$ ldd example
        linux-vdso.so.1 =>  (0x00007ffd565fd000)
        libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007fa81889c000)
        libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007fa81867e000)
        librt.so.1 => /lib/x86_64-linux-gnu/librt.so.1 (0x00007fa818475000)
        libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007fa81825f000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fa817e9a000)
        /lib64/ld-linux-x86-64.so.2 (0x00007fa818cf9000)
        libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007fa817b93000)
```

Dynamic linking on Linux can be undesirable if you wish to target older
machines as applications compiled aginst newer versions glibc are not
guaranteed to run against older versions.

You can examine Rust linking arguments with an option to rustc. Newlines have
been added for readability:

``` text
$ rustc example.rs -Z print-link-args
"cc"
    "-Wl,--as-needed"
    "-m64"
    [...]
    "-o" "example"
    "example.o"
    "-Wl,--whole-archive" "-lmorestack" "-Wl,--no-whole-archive"
    "-Wl,--gc-sections"
    "-pie"
    "-nodefaultlibs"
    [...]
    "-Wl,--whole-archive" "-Wl,-Bstatic"
    "-Wl,--no-whole-archive" "-Wl,-Bdynamic"
    "-ldl" "-lpthread" "-lrt" "-lgcc_s" "-lpthread" "-lc" "-lm" "-lcompiler-rt"
```

Arguments with a `-L` before them set up the linker search path and arguments
ending with `.rlib` are linking Rust crates statically into your application.
Neither of these are relevent for static linking so have been ommitted.

The first step in being able to statically link is to obtain an object file.
This can be achieved with `rustc --emit obj example.rs`, and creates a file
called `example.o`, which you can see being passed in the command line above -
rustc automatically deletes it when finished with it by default. As you now have
the object file, you should be able to run the link command obtained with
`print-link-args` to create perform the linking stage yourself.

In order to statically link, there are a number of changes you must make. Below
is the command required to perform a static link; we will go through them each
in turn.

``` text
$ rustc example.rs -Z print-link-args
"cc"
    "-static"
    "-m64"
    [...]
    "-o" "example"
    "example.o"
    "-Wl,--whole-archive" "-lmorestack" "-Wl,--no-whole-archive"
    "-Wl,--gc-sections"
    "-nodefaultlibs"
    [...]
    "-Wl,--whole-archive"
    "-Wl,--no-whole-archive"
    "-ldl" "-lpthread" "-lrt" "-lgcc_eh" "-lpthread" "-lc" "-lm" "-lcompiler-rt"
```

 - `-static` was added - this is the signal to the compiler to use a static
   glibc, among other things
 - `-Wl,--as-needed` was removed - this can be left in, but is unnecessary
   as it only applies to dynamic librares
 - `-pie` was removed - this is not compatible with static binaries
 - both `-Wl,-B*` options were removed - everything will be linked statically,
   so informing the linker of how certain libraries should be linked is not
   appropriate
 - `-lgcc_s` was changed to `-lgcc_eh` - `gcc_s` is the GCC support library,
   which Rust uses for unwinding support. This is only available as a dynamic
   library, so we must specify the static version of the library providing
   unwinding support.

By running this command, you will likely see some warnings like

``` text
warning: Using 'getpwuid_r' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
```

These should be considered carefully! They indicate calls in glibc which
*cannot* be statically linked without significant extra effort. An application
using these calls will find it is not as portable as 'static binary' would imply.
Rust supports targeting musl as an alternative libc to be able to fully
statically link these calls.

As we are confident that our code does not use these calls, we can now see the
fruits of our labour:

```
$ ldd example
        not a dynamic executable
```

This binary can be copied to virtually any 64-bit Linux machine and work
without requiring external libraries.
