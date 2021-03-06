# status

Show changed stages in the pipeline and mismatches either between the local
cache and local files, or between the local cache and remote cache.

## Synopsis

```usage
    usage: dvc status [-h] [-q | -v] [-j JOBS] [--show-checksums]
                      [-c] [-r REMOTE] [-a] [-T] [-d]
                      [targets [targets ...]]
    
    positional arguments:
      targets               Limit the scope to these stage files.
```

## Description

`dvc status` searches for changes in the pipeline, either showing which stages
have changed in the local workspace and must be reproduced (`dvc repro`), or
differences between the local cache and remote cache (meaning `dvc push` or
`dvc pull` should be run to synchronize them). The two modes, _local_ and 
_cloud_ are triggered by using the `--cloud` option:

Mode   | CLI Option | Description
-------|------------|----------------------------------
local  | _none_     | Comparisons are made between data files in the workspace and corresponding files in the local cache (`.dvc/cache`) 
remote | `-c`  | Comparisons are made between the local cache, and a cache stored elsewhere.  Remote caches are defined using the `dvc remote` command.

DVC determines data and code files to compare by analyzing all stage files in
the current workspace (`--all-branches` and `--all-tags` in the `cloud` mode
compare multiple workspaces - across all branches or tags). The comparison can be
limited to one or more stages by listing the stage file(s) as `targets`. Changes
are reported only against the named target stage or stages. When combined with
the `--with-deps` option, a search is made for changes in other stages that
affect the target stage.

In the `local` mode, changes are detected through the checksum of every file
listed in every stage file in the pipeline against the corresponding file in the
filesystem. The output indicates the detected changes, if any.  If no
differences are detected, `dvc status` prints this message:

```dvc
    $ dvc status
    Pipeline is up to date. Nothing to reproduce.
```

This says that no differences were detected, and therefore that no stages would
be rerun if `dvc repro` were executed.

If instead differences have been detected, `dvc status` lists those changes. 
For each stage with differences, the _dependencies_ and/or _outputs_ that differ
are listed.  For each item listed, either the file name or the checksum is
shown, and additionally a status word is shown describing the change:

* For the local workspace:
    * _changed_ means the named file has changed
* For comparison against a remote cache:
    * _new_ means the file exists in the local cache but not the remote cache
    * _deleted_ means the file does not exist in the local cache, and exists in
      the remote cache

For the _changed_ case, the `dvc repro` command is indicated.

For either the _new_ and _deleted_ cases, the local cache (subset of it, that is
determined by the active workspace) is different from the remote cache. 
Bringing the two into sync requires  `dvc pull` or `dvc push` to synchronize the
DVC cache. For the typical process to update workspaces, see [Share Data And
Model Files](/doc/use-cases/share-data-and-model-files).
 
## Options

* `-d`, `--with-deps` finds changes by tracking dependencies to the named target
  stage.  This option only has effect when one or more target stages are named. 
  By traversing the dependencies, DVC searches backward through the pipeline
  from the named target(s).  This means DVC will not show changes occurring
  later in the pipeline than the named target(s).  Applies whether or not
  `--cloud` is specified.

* `-v`, `--verbose` displays detailed tracing information from executing the
  `dvc status` command.

* `-q`, `--quiet` do not write anything to standard output. Exit with 0 if
  pipeline is up to date, otherwise 1.

* `-c`, `--cloud` enables comparison against a remote cache.  If no `--remote`
  option has been given, DVC will compare against the default remote cache. 
  Otherwise the comparison will be against the remote specified in the
  `--remote` option.  Using `--cloud` enables the remaining options.

* `-r REMOTE`, `--remote REMOTE` specifies which remote cache (see `dvc remote
  list`) to compare against.  The argument, `REMOTE`, is a remote name defined
  using the `dvc remote` command.   Applies only if `--cloud` is specified.

* `-a`, `--all-branches` compares cache content against all git branches. 
  Instead of checking just the currently checked out workspace, it checks
  against all other branches of this workspace.  The corresponding branches are
  shown in the status output.  Applies only if `--cloud` is specified.

* `-T`, `--all-tags`  compares cache content against all git tags.  Both the
  `--all-branches` and `--all-tags` options cause DVC to check more than just
  the currently checked out workspace.  The corresponding tags are shown in the
  status output.  Applies only if `--cloud` is specified.

* `--show-checksums`  shows the DVC checksum for the file, rather than the file
  name.  Applies only if `--cloud` is specified.

* `-j JOBS`, `--jobs JOBS` specifies the number of jobs DVC can use to retrieve
  information from remote servers.  This only applies when the `--cloud` option
  is used.

## Example: Simple usage

```dvc
$ dvc status

  bar.dvc
          outs
                  changed:  bar
          deps
                  changed:  foo
  foo.dvc
          outs
                  changed:  foo
```

This shows that for `bar.dvc` the dependency, `foo`, has changed, and the
output, `bar` has changed.  Likewise for `foo.dvc` the dependency `foo` has
changed, but no output has changed.

## Example: Dependencies

```dvc
$ vi code/featurization.py
... edit the code
$ dvc status model.p.dvc 
Pipeline is up to date. Nothing to reproduce.
$ dvc status model.p.dvc --with-deps
matrix-train.p.dvc
	deps
		changed:  code/featurization.py
```

If the `dvc status` command is limited to a target that had no changes, result
shows no changes.  By adding `--with-deps` the change will be found, so long as
the change is in a preceding stage.

## Example: Remote comparisons

```dvc
$ dvc remote list
rcache	/Volumes/Extra/dvc/classify/rcache
$ dvc status --cloud --remote rcache
Preparing to collect status from /Volumes/Extra/dvc/classify/rcache
[##############################] 100% Collecting information
	new:      data/model.p
	new:      data/eval.txt
	new:      data/matrix-train.p
	new:      data/matrix-test.p
$ dvc status --cloud --remote rcache --show-checksums
Preparing to collect status from /Volumes/Extra/dvc/classify/rcache
[##############################] 100% Collecting information
	new:      f0a6e3eed7c7c1a1c707da2c1673ca72
	new:      d6b228f7904bd200d4eb643fe0e8efd8
	new:      f506aa14271f793ffd7eca113f5920cd
	new:      9c0b1f5c3560b6a2838b3fbcd7d72665
```

The output shows where the location of the remote cache as well as any
differences between the local cache and remote cache.
