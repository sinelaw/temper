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
