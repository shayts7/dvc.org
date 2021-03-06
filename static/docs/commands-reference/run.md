# run

Generate a stage file from a given command and execute the command.

## Synopsis

```usage
    usage: dvc run [-h] [-q | -v] [-d DEPS] [-o OUTS]
                   [-O OUTS_NO_CACHE] [-M METRICS_NO_CACHE]
                   [-m METRICS] [-f FILE] [-c CWD]
                   [--no-exec] [-y] [--overwrite-dvcfile]
                   [--ignore-build-cache] [--remove-outs]
                   [--no-commit]
                   command
    
    positional arguments:
      command               Command to execute.
```

## Description

`dvc run` provides an interface to build a computational graph (aka pipeline).
It's a way to describe commands, data inputs and intermediate results that went
into a model (or other data results). By explicitly specifying a list of
dependencies (with `-d` option) and outputs (with `-o`, `-O`, or `-M` options)
DVC can connect individual stages (commands) into a direct acyclic graph (DAG).
`dvc repro` provides an interface to check state and reproduce this graph
later. This concept is similar to the one of the `Makefile` but DVC captures
data and caches data artifacts along the way. Check this 
[example](/doc/get-started/example-pipeline) to learn more and try to build a
pipeline.

By default, unless `-f` options is specified, stage file name generated is
`<file>.dvc` where `<file>` is the file name of the first output (`-o`, `-O`,
or `-M` option). If neither `-f`, nor outputs (with `-o`, `-O`, `-M` options)
are specified stage name defaults to `Dvcfile`. 

Since `dvc run` provides a way to build a graph of computations, using
dependencies and outputs to connect different stages it checks computational
graph integrity properties before creating a new stage. For example, for every
output there should be only one stage that explicitly specifies it. There
should be no cycles, etc.

## Options

* `-d`, `--deps` - specify a file or a directory the stage depends on. Multiple
dependencies can be specified like this: `-d data.csv -d process.py`.
Usually, each dependency is a file or a directory with data, or a code file, or
a configuration file. DVC is building a computation graph and this list of
dependencies is a way to connect different stages with each other. Also, when
you run `dvc repro` to reproduce a stage (or when stage is reproduced due to
recursive dependency), list of dependencies helps DVC to analyze if one or more
of the dependencies listed have changed and running the stage again is
required. Special case is when no dependencies are specified. Stage file
without dependencies is considered always _changed_ and `dvc repro` always
executes it.

* `-o`, `--outs` - specify a file or a directory that are results of running
the command. Multiple outputs can be specified like this: `-o model.pkl -o
output.log`. DVC is building a computation graph and this list of outputs
(along with dependencies described above) is a way to connect different stages
with each other. DVC takes all output files and directories under its control
and will put them into the cache (this is similar to what's happening when you
run `dvc add`).

* `-O`, `--outs-no-cache` - the same as `-o` except outputs are not put
automatically under DVC control. It means that they are not cached, and it's
up to a user to save and version control them. Usually, it's useful if outputs
are small enough to be put into Git or other underlying version control system,
or these files are not of any interest and there is no requirement to save and
share them. 

* `-m`, `--metrics` - another kind of output files. It is usually a
small human readable file (JSON, CSV, text, whatnot) with some numbers or other
meta-information that describes a model or other outputs. Check `dvc metrics`
to learn more about tracking metrics and comparing them across different
model or experiment versions. Metrics are not cached (put under DVC control),
you should see `cache: false` in the stage file. Since these files are small
enough it's beneficial to use Git or any other underlying regular version
control system to track them.

* `-M`, `--metrics-no-cache` - the same as `-m` except files are not put
automatically under DVC control. In case of metrics it's pretty usual because
metric files are small enough to be put into Git or other underlying version
control system. See also the difference between `-o` and `-O` options.

* `-f`, `--file` - specify stage file name. By default stage file name
generated is `<file>.dvc` where `<file>` is file name of the first output (
`-o`, `-O`, or `-M` option). If neither `-f`, nor outputs (with `-o`, `-O`,
`-M`) are specified stage name defaults to `Dvcfile`. Stage file name may not
contain subdirectories, use `-c` options to change location of the stage file. 

* `-c`, `--cwd` - changes a current working directory before running the
command. All dependencies and outputs will be expanded relative to that 
directory. Thus `dvc run --cwd <DIR> [...]` is equivalent to 
```dvc
    $ pushd <DIR>
    $ dvc run [...]
    $ popd
```
* `--no-exec` - create a stage file, do not run the command specified do not
take dependencies or outputs under DVC control. On the stage file level `md5`
hash sums will be empty. They will be populated next time this stage is
actually executed. This command is useful, if for example, you need to build
a pipeline (computational graph) first, and then run it all at once.

* `-y`, `--yes` - deprecated, use `--overwrite-dvcfile` instead.

* `--overwrite-dvcfile` - overwrite an existing stage file (the same file name
which is determined by the logic described in the `-f` option) without asking
for confirmation.

* `--ignore-build-cache` - if an exactly the same (same list of outputs and
inputs, the same command to run) stage file exists and it has been already
executed, and is up to date, and `--overwrite-dvcfile` is specified `dvc run`
won't execute the command again by default (thus, `build cache`). This option
gives a way to forcefully run the command anyway. It's useful, if command is
considered a non-deterministic for some reason (it means it produces different
outputs from the same list of inputs). 

* `--remove-outs` - it removes stage outputs before running the command. If
`--no-exec` specified outputs are removed anyway. See `dvc remove` as well for
more details.     

* `--no-commit` - doesn't save outputs to cache. Useful when running different
experiments and you don't want to fill up your cache with temporary files.
Use `dvc commit` when you are ready to save your results to cache.

## Examples

* A trivial example to play with, try different set of options to see how they
work. You don't need any actual data or scripts to play with this example:

```dvc
   $ mkdir example
   $ cd example
   $ git init
   $ dvc init 
   $ mkdir data
   $ dvc run -d data -o metric -f metric.dvc "echo '1' >> metric"
   
   Running command:
   	echo 'a' >> metric
   Adding 'metric' to '.gitignore'.
   Saving 'metric' to cache '.dvc/cache'.
   Saving information to 'metric.dvc'.
   
   To track the changes with git run:
   
   	git add .gitignore metric.dvc
```

* Execute a Python script as a DVC pipeline step. Stage file name is not
specified, so a `model.p.dvc` stage file is created:

```dvc
    # Train ML model on the training dataset. 20180226 is a seed value.
    $ dvc run -d matrix-train.p -d train_model.py \
              -o model.p \
              python train_model.py matrix-train.p 20180226 model.p
```

* Execute an R script as s DVC pipeline step:

```dvc
    $ dvc run -d parsingxml.R -d Posts.xml \
              -o Posts.csv \
              Rscript parsingxml.R Posts.xml Posts.csv
```

* Extract an XML file from an archive to the `data/` folder:

```dvc
    $ mkdir data
    $ dvc run -d Posts.xml.zip \
              -o data/Posts.xml \
              -f extract.dvc \
              unzip Posts.xml.zip -d data/
```
