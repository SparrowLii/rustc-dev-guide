# Parallel Compilation

As of <!-- date-check --> May 2022, The only stage of the compiler
that is already parallel is codegen. The nightly compiler implements query evaluation,
but there is still a lot of work to be done. The lack of parallelism at other stages
also represents an opportunity for improving compiler performance. One can try out the current
parallel compiler work by enabling it in the `config.toml`.

These next few sections describe where and how parallelism is currently used,
and the current status of making parallel compilation the default in `rustc`.

The underlying thread-safe data-structures used in the parallel compiler
can be found in the `rustc_data_structures::sync` module. Some of these data structures
use the `parking_lot` crate as well.

## Codegen

There are two underlying thread safe data structures used in code generation:

- `Lrc`
    -  Which is an [`Arc`][Arc] if `parallel_compiler` is true, and a [`Rc`][Rc]
       if it is not.
- `MetadataRef` -> [`OwningRef<Box<dyn Erased + Send + Sync>, [u8]>`][OwningRef]
    - This data structure is specific to `rustc`.

During [monomorphization][monomorphization] the compiler splits up all the code to
be generated into smaller chunks called _codegen units_. These are then generated by
independent instances of LLVM running in parallel. At the end, the linker
is run to combine all the codegen units together into one binary. This process
occurs in the `rustc_codegen_ssa::base` module.

## Query System

The query model has some properties that make it actually feasible to evaluate
multiple queries in parallel without too much of an effort:

- All data a query provider can access is accessed via the query context, so
  the query context can take care of synchronizing access.
- Query results are required to be immutable so they can safely be used by
  different threads concurrently.

When a query `foo` is evaluated, the cache table for `foo` is locked.

- If there already is a result, we can clone it, release the lock and
  we are done.
- If there is no cache entry and no other active query invocation computing the
  same result, we mark the key as being "in progress", release the lock and
  start evaluating.
- If there *is* another query invocation for the same key in progress, we
  release the lock, and just block the thread until the other invocation has
  computed the result we are waiting for. This cannot deadlock because, as
  mentioned before, query invocations form a DAG. Some threads will always make
  progress.

## Rustdoc

As of <!-- date-check--> May 2022, there are still a number of steps
to complete before rustdoc rendering can be made parallel. More details on
this issue can be found [here][parallel-rustdoc].

## Current Status

As of <!-- date-check --> May 2022, work on explicitly parallelizing the
compiler has stalled. There is a lot of design and correctness work that needs
to be done.

These are the basic ideas in the effort to make `rustc` parallel:

- There are a lot of loops in the compiler that just iterate over all items in
  a crate. These can possibly be parallelized.
- We can use (a custom fork of) [`rayon`] to run tasks in parallel. The custom
  fork allows the execution of DAGs of tasks, not just trees.
- There are currently a lot of global data structures that need to be made
  thread-safe. A key strategy here has been converting interior-mutable
  data-structures (e.g. `Cell`) into their thread-safe siblings (e.g. `Mutex`).

[`rayon`]: https://crates.io/crates/rayon

As of <!-- date-check --> May 2022, much of this effort is on hold due
to lack of manpower. We have a working prototype with promising performance
gains in many cases. However, there are two blockers:

- It's not clear what invariants need to be upheld that might not hold in the
  face of concurrency. An auditing effort was underway, but seems to have
  stalled at some point.

- There is a lot of lock contention, which actually degrades performance as the
  number of threads increases beyond 4.

Here are some resources that can be used to learn more (note that some of them
are a bit out of date):

- [This IRLO thread by Zoxc, one of the pioneers of the effort][irlo0]
- [This list of interior mutability in the compiler by nikomatsakis][imlist]
- [This IRLO thread by alexchricton about performance][irlo1]
- [This tracking issue][tracking]

[irlo0]: https://internals.rust-lang.org/t/parallelizing-rustc-using-rayon/6606
[imlist]: https://github.com/nikomatsakis/rustc-parallelization/blob/master/interior-mutability-list.md
[irlo1]: https://internals.rust-lang.org/t/help-test-parallel-rustc/11503
[tracking]: https://github.com/rust-lang/rust/issues/48685
[monomorphization]: backend/monomorph.md
[parallel-rustdoc]: https://github.com/rust-lang/rust/issues/82741
[Arc]: https://doc.rust-lang.org/std/sync/struct.Arc.html
[Rc]: https://doc.rust-lang.org/std/rc/struct.Rc.html
[OwningRef]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_data_structures/owning_ref/index.html
