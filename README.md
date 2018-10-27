# The Temper Build System

Temper will be (once implemented) a new incarnation of [Buildsome](https://github.com/buildsome/buildsome).

In Temper, dependencies are automatically inferred. When an output file (or *build target*) is built, Temper will track
its file system activity and determine which input files are dependencies of this target.

## Design

Hopefully uncoupled parts:

1. Build rules oracle
2. File system hook
3. History cache
4. Job runner


## Build Rules

Build rules describe how to build each target file. For any given target file the build rules will say which command
must be invoked to create that target.

For example (assuming Makefile syntax), the rule:

    foo.o:
        gcc -c foo.c -o foo.o

Is a build rule.

Patterns are supported, for example, a rule can say any file `%.o` is built from `%.c` using: `gcc -c %.c -o %.o` (where
`%` is a pattern matching arbitrary strings).

## File System Hook

Every process run by Temper is tracked for file system activity. If the process tries to read some file, Temper will
record it as a dependency (or "input") of the current target file. Inputs themselves can be targets (such as
auto-generated code), in which case Temper will first build the input before allowing the running process to actuall
read it. In other words, Temper satisfies auto-generated build requirements on demand.

The file system hook comunicates with an arbitrary server. The API of the file system hook is:

- Run a process
- Whenever the process tries to access a file, make a call to the server describing the access.
  The running process is paused until the server sends a response.

## History Cache

The history cache serves to avoid running build rule commands for outputs if we know from past experience what the
output should be (given the current state of the file system). Also, the history cache tells us which dependencies were
detected last time a given output was created, which allows parallelizing work on a target.

The history cache answers the question: for a given command, from past experience, what inputs would lead to which
output files (and what would be in those files)? More specifically, given the current state of the file system, is there
a known output file which was created from exactly the current state of the inputs?

The history cache is populated every time a build rule is executed. Recorded information includes:

- Which files were accessed (served as inputs) to this command, and what was their state at the time (attributes,
  contents hash).
- What outputs were created (and their attributes, contents).


## Job Runner

The job runner drives the build process. It receives an initial list of targets to build. It checks with the build rules
to see how to build the targets. Then it consults the history cache and avoids actually executing the build commands if
possible. If running the command can't be avoided, it uses the file system hook to track dependencies and resolve them
on-the-fly.

The job runner should aim to:

1. Keep the number of running jobs close to the requested concurrency (while avoiding deadlocks).
2. Avoid pausing jobs for a long time while their dependencies are being satisfied (paused jobs use memory and other
   resources).
3. Build everything as quickly as possible.

The build rules imply a set of jobs (commands) that must be run. Those jobs in turn may turn out to depend on other
jobs. Thus, the jobs can be described as a directed acyclic graph (DAG).

Dependencies are detected at job execution time, so specifying dependencies is entirely optional. Once a certain build
target has been fully built, its dependency graph can be estimated for the next run.

## Example

For example of how dependencies are determined at run time, consider (Makefile syntax is used just for example):

    example_program:
        ld module.o -o example_program

    module.o:
        gcc source.c -c -o module.o

The first time `example_program` is being built, the job runner only knows what command needs to be run (`ld`), but the
dependencies are unknown. While the command is being executed, the file system hook finds out that the file `module.o`
is being access. The job runner can now be certain that `example_program` depends on `module.o`. The already runnning
`ld` is kept waiting while the dependency is satisfied. Since `module.o` itself appears in the build rules as a known
target, a new command is executed (`gcc`) and in the same fashion `source.c` is found to be a dependency of
`module.o`. Since `source.c` exists (it isn't a target itself), no further action is needed.

Let's assume only the contents of `source.c` change, and then the job is run again. The depedency graph for the last run
can be consulted and then `module.o` is built first, before the command for building `example_prgram` is executed.

In the above example none of the dependencies were specified explicitly, but users may choose to specify *some*
dependencies for documentation or as hints that allow the build process to be parallelized also on the first run, when
automatically inferred dependencies are not yet known.

## Dependencies and Job Running

If no dependencies are specified, and they all must be detected at run time, there are two cases:

1. First execution - nothing is known about the dependency tree. Each time the "next" job is executed while a whole
   bunch of other jobs are paused, blocking on a target file access that can't proceed until that target is
   built. Dependencies are learned and recorded in the history database.
2. Execution after the history database is populated - dependencies can be guessed (they may have changed due to changes
   in the source code, but a guess is possible), and "leaf" jobs can be executed first without holding up other jobs.

Concurrency in the second case is easy: build the leaves of the dependency tree first, in parallel. Then proceed up
the tree, repeating until the root is reached.

The first case, when nothing is known, is more problematic. We can't parallelize jobs if we don't know what they're
going to be. Jobs will be paused, blocking on access to other targets until they are built. There's no information in
advance about how long each job will be blocked and how many resources (e.g. memory) will be held up while they wait for
other targets to be build. The best we can do is use heuristics to limit the running jobs from consuming all the
system's resources (for example, a naive rule can be: if a job has being blocked for more than a certain amount of time,
it should be aborted and restarted later when all its dependencies so far have been built).

## Limiting Concurrency

### Why limit concurrency?

Once dependencies are known, the maximum parallelization of jobs is determined by how "entangled" the dependency graph
is.  If it's like a nice wide tree, with say 100 leaf nodes (jobs that have no further dependencies) we could run 100
jobs in parallel when starting to build. However, practical concerns may impose an upper limit on how many jobs can be
run concurrently: the amount of memory available to the system is a strong limit, but users may also want to allow the
system some CPU slack or to prevent overloading the storage device in order to allow other programs to run. For this
reason, we must allow limiting the parallelization.

### How concurrency can be limited

There are several types of jobs:

1. Explicitly requested by the user - the top-level targets, which imply the top-level jobs.
2. Explicit dependencies of other jobs - user-defined and parsed from the build rules.
3. Dynamically detected dependencies.
4. Assumed dependencies, guessed according to the history.

The following algorithm limits the concurrency:

    tokens <- number of concurrency tokens (= the concurrency limit)
    known_targets <- the top-level (root) targets
    inferred_targets <- empty list
    guessed_targets <- empty list
    while known_targets is not empty:
        cur_target <- pop head from known_targets
        explicit_deps <- check the build rules for any explicit dependencies of cur_target
        push explicit_deps onto known_targets
        guessed_deps <- check the history cache for any guessed dependencies of cur_target
        push guessed_deps onto guessed_targets
        if both explicit_deps and guessed_targets are empty:
            run the job for building cur_target, if any dependency is encountered,
            block on its completion and insert it into inferred_targets
