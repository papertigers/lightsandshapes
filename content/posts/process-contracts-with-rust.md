---
title: "Process contracts with Rust"
date: 2023-08-13
category: "systems programming"
tags: ["libcontract", "illumos", "rust", "ffi"]
---

![tailscale + omnios](/images/rust-contracts.png)

### Setting the stage

I recently had a FaceTime call with my friend [Dave](https://daveeddy.com)
discussing technical topics -- as we often do. This conversation in particular
was about Dave's desire to write some new Rust software to manage [SMF][smf]
services on his home server. The question at hand was, "how can we reliably
determine if a service has processes associated with it or not". The service
management facility (SMF) in illumos provides a property called
`startd/duration`. This property can be set to a few options, but the default is
*contract*.

Okay, but what is a contract? On illumos there is a filesystem called
`ctfs(4FS)` that provides an interface to the [contract sub-system][contract].
The man page does an excellent job summarizing what a "contract" provides:

> Process contracts allow processes to create a fault boundary around a
> set of subprocesses and observe events which occur within that boundary.

![logic contract layout](/images/logical_contract_example.svg#center)

In the figure above you can see that the contract creates a logical boundary
around some group of processes. Contracts can be nested in the sense that they
are layered, however, using the above diagram as an example you will not see
the PIDs of contract 12 when observing contract 11. This is intentional, as
contracts can be orphaned by the controlling process and inherited by another.
This behavior is easily observable with `ptree` and
`ctstat`:

```text
$ ptree -gc $$
6172   zsched
└─[process contract 2798: svc:/network/ssh:default]
  └─25661  /usr/sbin/sshd -R
    └─25664  /usr/sbin/sshd -R
      └─25666  -bash
        ├─25708  ctrun -l child ./a.out
        │ └─[process contract 2799: svc:/network/ssh:default]
        │   └─25710  ./a.out
        │     └─25712  ./a.out
        │       └─25713  ./a.out
        └─26325  ptree -gc 25666

$ ctstat -vi 2798 | rg member
        member processes:      25661 25664 25666 25708 26666 26667
$ ctstat -vi 2799 | rg member
        member processes:      25710 25712 25713
```

On the call we discussed what retrieving this member list of process IDs from a
contract in a Rust program might look like. Almost instantly I was drawn into
figuring out how to solve the proposed technical challenge laid out in front of
me!

### Why are contracts useful

Before I dive into solving the question at hand, I wanted to take the
opportunity to discuss at least one example of why contracts are useful. Let's
imagine that you have a service you are writing and you plan to eventually run
it under some service management tool such as SMF. It's the job of the service
manager to keep track of all the processes associated with your service no
matter what actions they might preform. One action your program may take is
calling `fork(2)` to create new children. But, what happens if your program
creates a child that also creates it's own child? Here's a C program that does
just that:

{{< filename file="example.c" >}}
```C
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>

int
main()
{
        pid_t child = -1,  grandchild = -1;
        pid_t parent = getpid();

        printf("parent: %d\n", parent);
        if ((child = fork()) == 0) {
                if ((grandchild = fork()) == 0) {
                        pause();
                } else {
                        printf("grandchild: %d\n", grandchild);
                        pause();
                }
        } else {
                printf("child: %d\n", child);
                waitpid(child, NULL, WCONTINUED);
                pause();
        }
}
```

{{< note >}}
This toy C program elides safety and error checking. It's meant to provide a
concise example.
{{< /note >}}

In the above example we end up with a program where there is a parent, child,
and grandchild relationship. If we run this program we can see that laid out for
us by `ptree`:

```text
$ ./a.out
parent: 860
child: 861
grandchild: 862
^Z
[1]+  Stopped                 ./a.out
$ ptree -g 860
6172   zsched
└─8507   /sbin/init
  └─13577  /usr/sbin/sshd
    └─25661  /usr/sbin/sshd -R
      └─25664  /usr/sbin/sshd -R
        └─25666  -bash
          └─860    ./a.out
            └─861    ./a.out
              └─862    ./a.out
```

If we pretend that the bash process is our service manager for a moment -- what
happens if we have the child (the middle-process) exit leaving only the parent
and grandchild?

```text
$ bg
[1]+ ./a.out &
$ kill 861
$ ptree -g 25666
6172   zsched
└─8507   /sbin/init
  └─13577  /usr/sbin/sshd
    └─25661  /usr/sbin/sshd -R
      └─25664  /usr/sbin/sshd -R
        └─25666  -bash
          ├─860    ./a.out
          └─1435   ptree -g 25666

$ ptree -g 862
6172   zsched
└─8507   /sbin/init
  └─862    ./a.out
```

Oops! Now it looks like our service manager just lost track of pid 862 because
it got orphaned and reparented to init. If we asked the service manager to show
us all of the processes associated with our service we wouldn't even be aware
that this process was running. Worse yet, if we wanted to tell our service
manager to stop this service it would no longer know it needed to kill the
grandchild in the teardown phase.

We can use `ctrun` on illumos to put the service in a contract allowing the
controlling process to keep track of all the children that were created. Here's
an example of what that looks like:

{{< highlight text "hl_lines=13-16 26-28">}}
$ ctrun -l child ./a.out &
[1] 2714
parent: 2717
child: 2719
grandchild: 2721
$ ptree -gc $$
6172   zsched
└─[process contract 2798: svc:/network/ssh:default]
  └─25661  /usr/sbin/sshd -R
    └─25664  /usr/sbin/sshd -R
      └─25666  -bash
        ├─2714   ctrun -l child ./a.out
        │ └─[process contract 2800: svc:/network/ssh:default]
        │   └─2717   ./a.out
        │     └─2719   ./a.out
        │       └─2721   ./a.out
        └─2795   ptree -gc 25666
$ kill 2719
$ ptree -gc $$
6172   zsched
└─[process contract 2798: svc:/network/ssh:default]
  └─25661  /usr/sbin/sshd -R
    └─25664  /usr/sbin/sshd -R
      └─25666  -bash
        ├─2714   ctrun -l child ./a.out
        │ └─[process contract 2800: svc:/network/ssh:default]
        │   ├─2717   ./a.out
        │   └─2721   ./a.out
        └─2838   ptree -gc 25666
{{< /highlight >}}

Great, now our pretend service manager would be able to view the contract's
process members to keep track of which processes are associated with our
service. With this information in hand, let's finally look at how we can view a
contracts members from a Rust program.

{{< note >}}
This post assumes some prior knowledge of Rust.
{{< /note >}}

### Interfacing with libcontract

Illumos provides a committed library interface called
[`libcontract(3LIB)`][libcontract]. While looking through the various associated
man pages I came across `ct_pr_status_get_members` -- a function enabling the
retrieval of process IDs belonging to the members associated with the process
contract. Well, isn't that precisely what we wanted to do? My first thought was
to define the function in Rust through an FFI interface, much like I did
[here][illumos-priv] for `priveleges(5)`. Instead, we can do better by
automatically generating Rust FFI bindings to the C library using [bindgen][].

Let's start by creating a new library crate, and adding `bindgen` to to the
build-dependencies:

```text
$ cargo new --lib contract-sys
$ cd contract-sys
$ cargo add --build bindgen
```

Next we will create the wrapper that tells bindgen where to look:

{{< filename file="wrapper.h" >}}
```C
#include <libcontract.h>
```

Then we create a build.rs file with the following:
{{< filename file="build.rs" >}}

```rust
extern crate bindgen;

use std::env;
use std::path::PathBuf;

fn main() {
    println!("cargo:rustc-link-lib=contract");
    println!("cargo:rerun-if-changed=wrapper.h");

    let bindings = bindgen::Builder::default()
        .header("wrapper.h")
        .generate()
        .expect("Unable to generate bindings");

    let out_path = PathBuf::from(env::var("OUT_DIR").unwrap());
    bindings
        .write_to_file(out_path.join("bindings.rs"))
        .expect("Couldn't write bindings!");
}
```

Finally we need to tell lib.rs to use the automatically generated files:
{{< filename file="src/lib.rs" >}}
```rust
#![allow(non_upper_case_globals)]
#![allow(non_camel_case_types)]
#![allow(non_snake_case)]

include!(concat!(env!("OUT_DIR"), "/bindings.rs"));
```

Now when we type `cargo build`, bindgen will run as a part of the build phase to
first create FFI definitions for the types found in our wrapper.h file.
{{< note >}}
The **contract-sys** crate has since been published to
[crates.io](https://crates.io/crates/contract-sys).
{{< /note >}}


### Using contract-sys

We have our bindings, and now it's time to use them! Create a new project and
add our dependency:

```text
$ cargo new contract-example
$ cd contract-example
$ cargo add contract-sys libc anyhow
```

If we pull back up the man page for `ct_pr_status_get_members` we see that it
takes three arguments:

```C
int ct_pr_status_get_members(ct_stathdl_t stathdl, pid_t **pidpp, uint_t *n);
```

The initial parameter to this function is a contract handle, obtainable
according to the man page by invoking the `ct_status_read` function:

```C
int ct_status_read(int fd, int detail, ct_stathdl_t *stathdlp);
```
> #### Arguments
>
> 1. `int_fd`: The fd to the file in ctfs where the data is being read from.
> 2. `int detail`:  The amount of detail returned can be large, therefore we
      can request different levels of detail with various flag values.
> 3. `ct_stathdl_t *stathdlp`: In standard C fashion, this is a pointer to the
      object handle that will be initalized by the function.


As outlined by the man page we can find the file we are interested in by
looking at `/system/contract/<type>/<id>`. For our purposes `id` will be
`status`.
{{< note >}}
   Each file is a symbolic link to the
   type-specific directory for that contract, that is
   `/system/contract/all/<id>` points to `/system/contract/<type>/<id>`.
{{< /note >}}

Let's start by accepting a contract id as an argument to our program, and using
it to open the corresponding status file:

{{< filename file="src/main.rs" >}}
```rust
use anyhow::{Context, Result};
use std::fs;

fn main() -> Result<()> {
    let args: Vec<String> = std::env::args().collect();
    let cid = match args.iter().nth(1) {
        Some(cid) => cid,
        None => {
            eprintln!("Usage: {} cid", args.first().unwrap());
            std::process::exit(1);
        }
    };

    let file = fs::File::open(format!("/system/contract/all/{cid}/status"))
        .context("failed to open contract status file")?;

    Ok(())
}
```

Now we need to make some storage space for our `ct_stathdl_t`. The proper way to
do this in Rust is to use the `MaybeUninit<T>` type - which is used to store a
type that may or may not be initialized. Note that this type was auto generated
for us when we built the `contract-sys` crate with bindgen:

```rust
let mut handle = MaybeUninit::<contract_sys::ct_stathdl_t>::uninit();
```

Then we call `ct_status_read` passing `CTD_ALL`, so that we retrieve all
possible data:

{{< highlight rust "hl_lines=5 13 22">}}
unsafe {
    if contract_sys::ct_status_read(
        file.as_raw_fd(),
        contract_sys::CTD_ALL as i32,
        handle.as_mut_ptr(),
    ) != 0
    {
        bail!("failed to read contract status");
    };
}

// handle should be initialized now
let handle = unsafe { handle.assume_init() };

// we can close the file now
drop(file);

// do something with our handle

// free our handle
unsafe {
    contract_sys::ct_status_free(handle);
}
{{< /highlight >}}

I've marked key lines that require our attention. Initially, we acquire a
mutable pointer to the handle, which will hold data subsequent to a successful
FFI call. Following that, we must indicate to the compiler that the data has
been initialized. Lastly, it's vital we release the allocated resources before
the program exits.

Looking back at the definition for `ct_pr_status_get_members` we need to pull
the same `MaybeUninit<T>` trick to make space for the pointer to the array of
process IDs:

```rust
let mut numpids = 0;
let mut pids = MaybeUninit::<*mut libc::pid_t>::uninit();

unsafe {
    if contract_sys::ct_pr_status_get_members(handle, pids.as_mut_ptr(), &mut numpids) != 0 {
        bail!("failed to get contract members");
    };
}
```

We are on the cusp of having a functioning program. All that is left to do now
is print the PIDs to stdout. That should be simple enough, right? Let's stuff
all the pids from the C array into a `Vec<pid_t>`:

```rust
let pids =
    unsafe { Vec::from_raw_parts(pids.assume_init(), numpids as usize, numpids as usize) };
println!("Here are the pids in the contract:\n {pids:#?}");
```

The moment of truth:

```text
$ cargo r -- 2621
   Compiling contract-example v0.1.0 (/home/link/src/contract-example)
    Finished dev [unoptimized + debuginfo] target(s) in 0.55s
     Running `target/debug/contract-example 2621`
Here are the pids in contract:
 [
    11623,
    4874,
    4873,
    4871,
]
Abort (core dumped)
```

![unsafe rust](/images/unsafe_rust.svg#center)

Uh oh!! That's not ideal, we got the PIDs in the contract but we also aborted.

### Debugging what went wrong

Unsafe Rust isn't named 'unsafe' for no reason! There are numerous ways
one could debug what went wrong, but given that we are on an illumos-based
system, let's use some of the exceptional tooling given to us by the OS.

Notice how in the terminal output above we generated a core dump. One of my
favorite aspects of the illumos community is our strong emphasis on post mortem
debugging. Let's fire up `mdb(1)` and see what sticks out to us.

We start by checking the status of the core file, enabling symbol demangling,
and dumping out the stack trace:

```text
$ mdb core
Loading modules: [ libumem.so.1 libc.so.1 libnvpair.so.1 ld.so.1 ]
> ::status
debugging core file of contract-example (64-bit) from rustdev
file: /home/link/src/contract-example/target/debug/contract-example
initial argv: target/debug/contract-example 2621
threading model: native threads
status: process terminated by SIGABRT (Abort), pid=15772 uid=1000 code=-1
> $G
C++ symbol demangling enabled
> $C
fffffbffffdeec50 libc.so.1`_lwp_kill+0xa()
fffffbffffdeec80 libc.so.1`raise+0x22(6)
fffffbffffdeec90 libumem.so.1`umem_do_abort+0x44()
fffffbffffdeed90 libumem.so.1`umem_err_recoverable+0xfe(fffffbffeedf7251)
fffffbffffdeede0 libumem.so.1`process_free+0xd4(713fa8, 1, 0)
fffffbffffdeee00 libumem.so.1`umem_malloc_free+0x1d(713fa8)
fffffbffffdeee80 <alloc::alloc::Global as core::alloc::Allocator>::deallocate::h47fa7257c9b4887c+0x6e()
fffffbffffdeeed0 <alloc::raw_vec::RawVec<T,A> as core::ops::drop::Drop>::drop::h6608692f52072a50+0x53()
fffffbffffdeeef0 core::ptr::drop_in_place<alloc::raw_vec::RawVec<i32>>::h1130edfbfc6daa72+0x11()
fffffbffffdeef20 core::ptr::drop_in_place<alloc::vec::Vec<i32>>::h4d2a08b20ad5db92+0x39()
fffffbffffdef3f0 contract_example::main::hdb086a8fdc0ab3bb+0x66d()
fffffbffffdef410 core::ops::function::FnOnce::call_once::hb57ae0e3d15f52e6+0xe()
fffffbffffdef440 std::sys_common::backtrace::__rust_begin_short_backtrace::h189237a4d6c478cb+0x11()
fffffbffffdef470 std::rt::lang_start::{{closure}}::hd3ba9fa9f64c423a+0x14()
fffffbffffdef540 std::rt::lang_start_internal::h23b82abee56a0268+0x2cc()
fffffbffffdef590 std::rt::lang_start::hf786f117a49d240f+0x37()
fffffbffffdef5a0 main+0x21()
fffffbffffdef5d0 _start_crt+0x87()
fffffbffffdef5e0 _start+0x18()
```

There are a few things that stand out to me here. First off we died to a
`SIGABRT` and second we can see from the stack trace that we are calling the
`Drop` method on the `Vec<i32>` that we allocated. It makes sense that after
the `Vec` leaves scope we would call the `Drop` method.

What's also cool about this stack trace is we can see [`libumem`][libumem] is on
the scene. Libumem is a powerful object-caching memory allocator. The library
provides extensive debugging support that we can take advantage of.

To turn on some of the debugging features we can set the `UMEM_DEBUG`
environment variable as well as `UMEM_LOGGING`. Both of these variables are
described in [umem_debug(3MALLOC)][umem_debug]:

```text
$ UMEM_LOGGING=transaction UMEM_DEBUG=default ./target/debug/contract-example 2621
Here are the pids in the contract:
 [
    17181,
    4874,
    4873,
    4871,
]
Abort (core dumped)
```

With a new core file generated let's hop back over to `mdb`:

{{< highlight text "hl_lines=36 56">}}
$ mdb core
Loading modules: [ libumem.so.1 libc.so.1 libnvpair.so.1 ld.so.1 ]
> $G
C++ symbol demangling enabled
> $C
fffffbffffdf57a0 libc.so.1`_lwp_kill+0xa()
fffffbffffdf57d0 libc.so.1`raise+0x22(6)
fffffbffffdf57e0 libumem.so.1`umem_do_abort+0x44()
fffffbffffdf58e0 libumem.so.1`umem_err_recoverable+0xfe(fffffbffef257251)
fffffbffffdf5930 libumem.so.1`process_free+0xd4(9e4fc8, 1, 0)
fffffbffffdf5950 libumem.so.1`umem_malloc_free+0x1d(9e4fc8)
fffffbffffdf59d0 <alloc::alloc::Global as core::alloc::Allocator>::deallocate::h47fa7257c9b4887c+0x6e()
fffffbffffdf5a20 <alloc::raw_vec::RawVec<T,A> as core::ops::drop::Drop>::drop::h6608692f52072a50+0x53()
fffffbffffdf5a40 core::ptr::drop_in_place<alloc::raw_vec::RawVec<i32>>::h1130edfbfc6daa72+0x11()
fffffbffffdf5a70 core::ptr::drop_in_place<alloc::vec::Vec<i32>>::h4d2a08b20ad5db92+0x39()
fffffbffffdf5f40 contract_example::main::hdb086a8fdc0ab3bb+0x66d()
fffffbffffdf5f60 core::ops::function::FnOnce::call_once::hb57ae0e3d15f52e6+0xe()
fffffbffffdf5f90 std::sys_common::backtrace::__rust_begin_short_backtrace::h189237a4d6c478cb+0x11()
fffffbffffdf5fc0 std::rt::lang_start::{{closure}}::hd3ba9fa9f64c423a+0x14()
fffffbffffdf6090 std::rt::lang_start_internal::h23b82abee56a0268+0x2cc()
fffffbffffdf60e0 std::rt::lang_start::hf786f117a49d240f+0x37()
fffffbffffdf60f0 main+0x21()
fffffbffffdf6120 _start_crt+0x87()
fffffbffffdf6130 _start+0x18()
> ::status
debugging core file of contract-example (64-bit) from rustdev
file: /home/link/src/contract-example/target/debug/contract-example
initial argv: ./target/debug/contract-example 2621
threading model: native threads
status: process terminated by SIGABRT (Abort), pid=17181 uid=1000 code=-1
> ::umem_status
Status:         ready and active
Concurrency:    64
Logs:           transaction=1m
Message buffer:
free(9e4fc8): double-free or invalid buffer
stack trace:
libumem.so.1'umem_err_recoverable+0xd3
libumem.so.1'process_free+0xd4
libumem.so.1'umem_malloc_free+0x1d
contract-example'_ZN63_$LT$alloc..alloc..Global$u20$as$u20$core..alloc..Allocator$GT$10deallocate17h47fa7257c9b4887cE+0x6e
contract-example'_ZN77_$LT$alloc..raw_vec..RawVec$LT$T$C$A$GT$$u20$as$u20$core..ops..drop..Drop$GT$4drop17h6608692f52072a50E+0x53
contract-example'_ZN4core3ptr54drop_in_place$LT$alloc..raw_vec..RawVec$LT$i32$GT$$GT$17h1130edfbfc6daa72E+0x11
contract-example'_ZN4core3ptr47drop_in_place$LT$alloc..vec..Vec$LT$i32$GT$$GT$17h4d2a08b20ad5db92E+0x39
contract-example'_ZN12contract_example4main17hdb086a8fdc0ab3bbE+0x66d
contract-example'_ZN4core3ops8function6FnOnce9call_once17hb57ae0e3d15f52e6E+0xe
contract-example'_ZN3std10sys_common9backtrace28__rust_begin_short_backtrace17h189237a4d6c478cbE+0x11
contract-example'_ZN3std2rt10lang_start28_$u7b$$u7b$closure$u7d$$u7d$17hd3ba9fa9f64c423aE+0x14
contract-example'_ZN3std2rt19lang_start_internal17h23b82abee56a0268E+0x2cc
contract-example'_ZN3std2rt10lang_start17hf786f117a49d240fE+0x37
contract-example'main+0x21
contract-example'_start_crt+0x87
contract-example'_start+0x18

> 9e4fc8::whatis
9e4fc8 is 9e4f80+48, freed from umem_alloc_96:
            ADDR          BUFADDR        TIMESTAMP           THREAD
                            CACHE          LASTLOG         CONTENTS
          9e6b60           9e4f80    51b742f825851                1
                           94f028           85e600                0
                 libumem.so.1`umem_cache_free_debug+0x16e
                 libumem.so.1`umem_cache_free+0x48
                 libumem.so.1`umem_free+0xb4
                 libumem.so.1`process_free+0x111
                 libumem.so.1`umem_malloc_free+0x1d
                 libnvpair.so.1`nv_free_sys+0x1c
                 libnvpair.so.1`nv_mem_free+0x1e
                 libnvpair.so.1`nvp_buf_free+0x24
                 libnvpair.so.1`nvlist_free+0x5a
                 libcontract.so.1`ct_status_free+0x2a
                 contract_example::main::hdb086a8fdc0ab3bb+0x654
                 core::ops::function::FnOnce::call_once::hb57ae0e3d15f52e6+0xe
                 std::sys_common::backtrace::__rust_begin_short_backtrace::h189237a4d6c478cb+0x11
                 std::rt::lang_start::{{closure}}::hd3ba9fa9f64c423a+0x14
                 std::rt::lang_start_internal::h23b82abee56a0268+0x2cc

>
{{< /highlight >}}

The additional environment variables grant us access to some of mdb's powerful
dcmds, with the primary one being `::umem_status.` This `dcmd` not only displays
the status message but also the message buffer. The command's output distinctly
reveals a double-free issue. Our enabling of `UMEM_LOGGING=transaction` further
provides access to the transactional log through `addr::whatis.` What's neat
about this is the fact that we get a different stack trace compared to the one we
previously saw with `$C.` This secondary stack trace illustrates what originally
freed the object.

Notice the call to ``libcontract.so.1`ct_status_free``. That is the final call we
made to clean up the resources allocated by `ct_status_read`. The issue here is
that we called an unsafe `Vec<T>` function to create owned data from the raw
parts (note the Rust docs call out other invariants that we are not upholding).
As a result we end up with the library function freeing the data and `Vec<T>`
running its `Drop` method when the owning variable leaves scope. We can fix this
by using `slice::from_raw_parts` instead.

With that our final program looks like this:

{{< filename file="src/main.rs" >}}
```rust
use anyhow::{bail, Context, Result};
use std::{fs, mem::MaybeUninit, os::fd::AsRawFd};

fn main() -> Result<()> {
    let args: Vec<String> = std::env::args().collect();

    // get the first argument to the program
    let cid = match args.iter().nth(1) {
        Some(cid) => cid,
        None => {
            eprintln!("Usage: {} cid", args.first().unwrap());
            std::process::exit(1);
        }
    };

    // open the relevant file from ctfs
    let file = fs::File::open(format!("/system/contract/all/{cid}/status"))
        .context("failed to open contract status file")?;

    // safely create some storage for where the handle will be initialized
    let mut handle = MaybeUninit::<contract_sys::ct_stathdl_t>::uninit();

    // read the data into our handle
    unsafe {
        if contract_sys::ct_status_read(
            file.as_raw_fd(),
            contract_sys::CTD_ALL as i32,
            handle.as_mut_ptr(),
        ) != 0
        {
            bail!("failed to read contract status");
        };
    }

    // handle should be initialized now
    let handle = unsafe { handle.assume_init() };

    // we can close the file now
    drop(file);

    let mut numpids = 0;

    // create storage for a pointer to an array of pid_t
    let mut pids = MaybeUninit::<*mut libc::pid_t>::uninit();

    // get all of the members associated with the contract
    unsafe {
        if contract_sys::ct_pr_status_get_members(handle, pids.as_mut_ptr(), &mut numpids) != 0 {
            bail!("failed to get contract members");
        };
    }

    // provide a nice way for us to loop over the pids
    let pids = unsafe { std::slice::from_raw_parts(pids.assume_init(), numpids as usize) };
    println!("Here are the pids in contract {cid}:\n {pids:#?}");

    // cleanup the handle now that we are done
    unsafe {
        contract_sys::ct_status_free(handle);
    }

    Ok(())
}
```

```text
$ cargo r -- 2621
   Compiling contract-example v0.1.0 (/home/link/src/contract-example)
    Finished dev [unoptimized + debuginfo] target(s) in 0.53s
     Running `target/debug/contract-example 2621`
Here are the pids in contract 2621:
 [
    17181,
    4874,
    4873,
    4871,
]
```

Hooray! We successfully got the PIDs in the contract without crashing this time.
However, it seems a little clumsy to be doing all this manual unsafe programming
when we want to interact with `libcontract`. Rust crate authors typically
provide a safe interface to OS libraries. In part two we will discuss how to
create this safe interface, and how we can use it to enter a contract of our
own much like `ctrun` allows us to do.

### Closing thoughts

Contracts are an incredibly fascinating feature of illumos that I'm glad to
have invested time into exploring on a deeper level. Exposing operating system
C libraries in Rust is extremely easy to do but caution must be taken when
making calls across the FFI boundary. Without `rustc` and the type system to
provide a memory safe experience it can be challenging to get the semantics
correct. I chose to leave my error in this post since it provided an
opportunity to show how the writing process went organically, and it gave me an
excuse to show off some of the debugging facilities that we have in illumos. I
hope you will join me for part two but in the meantime if there are any other
Rust / illumos topics you would like me to dive into feel free to leave a comment
below with what you would like to learn more about.

[smf]: https://illumos.org/man/7/smf
[contract]: https://illumos.org/man/process.5
[libcontract]: https://illumos.org/man/3LIB/libcontract
[umem_debug]: https://illumos.org/man/3MALLOC/umem_debug
[illumos-priv]: https://github.com/TritonDataCenter/rust-illumos-priv/blob/26afc9d24f3c061645e11a2338c9e607e3c4fefc/src/ffi.rs
[bindgen]: https://github.com/rust-lang/rust-bindgen
[libumem]: https://illumos.org/man/3LIB/libumem
