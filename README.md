# Generator for missing Intel oneAPI Fortran library on macOS

## Rationale

In trying to use Intel oneAPI Fortran on macOS, [I recently found](https://community.intel.com/t5/Intel-Fortran-Compiler/Build-issue-with-2021-3-on-macOS-for-mac-ftn-alloc-dylib/td-p/1298844) that versions
greater than 2021.2 seem to not work. The bug presents at build time as:
```
ld: file not found: @rpath/for_mac_ftn_alloc.dylib for architecture x86_64
```

Investigations into the Fortran libraries in oneAPI show that this new
library, `for_mac_ftn_alloc.dylib`, is wanted by oneAPI as of 2021.3
in `libifcore.dylib` and in `libifcoremt.dylib`. You can see this with
`otool`:

```console
❯ otool -L libifcore.dylib | grep for_mac
	@rpath/for_mac_ftn_alloc.dylib (compatibility version 1.0.0, current version 1.11.0, weak)
❯ otool -L libifcoremt.dylib | grep for_mac
	@rpath/for_mac_ftn_alloc.dylib (compatibility version 1.0.0, current version 1.11.0, weak)
❯ otool -l libifcore.dylib | less
libifcore.dylib:
Load command 0
      cmd LC_SEGMENT_64
  cmdsize 632
  segname __TEXT
...
Load command 8
      cmd LC_VERSION_MIN_MACOSX
  cmdsize 16
  version 10.11
      sdk 10.13
Load command 9
      cmd LC_SOURCE_VERSION
  cmdsize 16
  version 0.0
Load command 10
          cmd LC_LOAD_WEAK_DYLIB
      cmdsize 56
         name @rpath/for_mac_ftn_alloc.dylib (offset 24)
   time stamp 2 Wed Dec 31 19:00:02 1969
      current version 1.11.0
compatibility version 1.0.0
Load command 11
          cmd LC_LOAD_DYLIB
      cmdsize 48
         name @rpath/libimf.dylib (offset 24)
   time stamp 2 Wed Dec 31 19:00:02 1969
...
```

The problem is, there is no `for_mac_ftn_alloc.dylib` in oneAPI or
anywhere else on my Mac. (It does not appear to be in gcc, Open MPI,
Xcode, etc.)

But this does present us with a possible hack/workaround. The
`for_mac_ftn_alloc.dylib` library is `LC_LOAD_WEAK_DYLIB` which, from my reading
of how this works, means [the library is allowed to not be present at run
time](http://www.dpldocs.info/experimental-docs/core.sys.darwin.mach.loader.LC_LOAD_WEAK_DYLIB.html).
Obviously it's not allowed to be missing at *build-time* though.

## Implementation

So, to work around this, a stub library was created. This library is very
boring:
```fortran
subroutine cws8vn9ud915dz907vl_()
end
```
It's essentially a "blank" library with a crazy procedure name that should never
be used that we are creating only to satisfy the need at `ld` time.

## How to build

To build, load up your Intel and run:
```console
❯ cmake --preset default && cmake --build --preset default
-- The Fortran compiler identification is Intel 2021.5.0.20211109
-- Detecting Fortran compiler ABI info
-- Detecting Fortran compiler ABI info - done
-- Check for working Fortran compiler: /opt/intel/oneapi/compiler/2022.0.0/mac/bin/intel64/ifort - skipped
-- Configuring done
-- Generating done
-- Build files have been written to: /Users/mathomp4/intellibhack/build
Scanning dependencies of target for_mac_ftn_alloc
[ 50%] Building Fortran object CMakeFiles/for_mac_ftn_alloc.dir/stub.F90.o
[100%] Linking Fortran shared library for_mac_ftn_alloc.dylib
[100%] Built target for_mac_ftn_alloc
```

## How to install

Once it builds, you should have a `for_mac_ftn_alloc.dylib` library in the
`build` directory. What you'll want to copy that file into the
`/opt/intel/oneapi/compiler/<version>/mac/compiler/lib`
directory. 

NOTE: I'm guessing you should build the library with the *exact same*
version of Intel oneAPI that you need the library for. I'm not good at shared
object libraries, etc. and with `otool` I see some hardcoded paths inside it.
