**Dask Design Document - 002**

# Dask CLI

**Authors**:

- Jacob Tomlinson (jtomlinson@nvidia.com)
- Matthew Rocklin
- Julia Signell

**Created**: 2022-07-20 (Last Updated: 2022-07-20)


## Abstract

We propose implementing a `dask` CLI tool that unites the variety of existing CLI tools that are littered across the Dask ecosystem. This CLI will cover new and existing functionality and will be extensible and support dispatching to other CLI tools.

## Motivation

Today we have the follow CLI tools:

- Distributed
  - `dask-scheduler` - Starts a scheduler
  - `dask-worker` - Starts a worker from CLI flags
  - `dask-spec` - Starts a worker from a JSON spec
  - `dask-ssh` - Launches a Dask cluster on remote machines using SSH
- Dask Control
  - `daskctl`
    - Allows list/scale/delete of Dask clusters that implement the `from_name` method for reconstruction of cluster objects.
    - Create Dask clusters from a specification file.
    - Also has an experimental TUI for viewing clusters and also has a CLI dashboard for systems where forwarding web ports is hard.
- Dask Foo
  - Many other `dask-foo` projects have a `dask-foo` command that launches a cluster via the CLI. See `dask-yarn` or `dask-ecs`.

This fragmentation leads to user confusion around all of these tools. It also means there are many inconsistencies in how these are implemented. Some use Click for simplified arg parsing, some use Rick for pretty terminal output, etc.

A primary motivation would be to unite everything under one CLI, then a secondary goal would be to improve consistency across tools.

## Requirements

- Users should be able to do everything they want via the `dask` CLI tool
- Starting processes should be done via subcommands, such as `dask scheduler` to start a scheduler.
- Projects like `dask-ctl` could contain it's functionality under a `dask cluster` namespace, for example `daskctl cluster list` would become `dask cluster list`.
- Useful debug information should be available at a command like `dask info` which prints out versions, sanitised config, etc for copy/paste into GitHub issues.

## Non-Goals

- Existing cluster CLI tools like `dask-yarn` that use `argparse` should still work without modification, just be mapped to a new namespace like `dask yarn` via an entrypoint.

## Detailed Description

## Implementation Details

Subprojects (including dask/distributed and dask/dask-ctl) can use entrypoints to add to the namespace.

- dask scheduler : we could have a dask command and split out to various subcommands, like conda.  This might make things a bit more extensible.
- dask worker: alias for `dask-worker`
- dask cluster list/start/stop/scale : It would be useful to manage long-running clusters from the command line (discussion on this topic on GitHub)
- dask job submit myscript.py –cluster <CLUSTER_NAME> : submits a script as if it were a function as with `client.submit`; returns a key for a Future that could be used to retrieve STDOUT via - dask job gather ${FUTURE_KEY} –cluster <CLUSTER_NAME>; could also include a –block or (--wait) flag for dask job submit to block and return STDOUT once task completes.
- dask –version
- dask info (what does this return?)
  - –version All of the versions for related libraries
    - this should use the `show_versions` function that was added in https://github.com/dask/dask/pull/9144
  - –config Config that is different from defaults
    - And anonymize anything that we shouldn’t send
- dask gateway/ssh should be able to extend
- dask worker shell < name>
- dask worker <name> –keepalive a mechanism for accessing the worker itself after Dask fails. That way you can inspect the state of the worker and try to figure out what went wrong.
- dask docs opens web browser pointing to docs

## Backward Compatibility

- All existing tools can still exist with their own commands in addition to being included in the `dask` CLI. But new tools going forwards should only be made via the new CLI.

## Alternatives

-

## Discussion
