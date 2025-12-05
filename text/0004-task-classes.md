# WDL task classes

This proposal introduces a "class" hint for tasks that allows users to change
implementation-defined configurations for multiple tasks at once. This is
particularly useful for users with specialized execution environments that
benefit from resource requests that are more specifically-tailored to the needs
of a task than the broad-strokes categorizations offered by task hints reserved
by the WDL specification.

## Problem description

Some Sprocket backends, such as
[LSF](https://sprocket.bio/configuration/backends/lsf.html), allow users to
configure execution differently for default tasks,
[`short`](https://github.com/openwdl/wdl/blob/wdl-1.2/SPEC.md#short_task) tasks,
and
[GPU/FPGA](https://github.com/openwdl/wdl/blob/wdl-1.2/SPEC.md#-gpu-and--fpga)
tasks. This allows, for example, CPU- and GPU-based tasks to be submitted to
different queues. However, these are not the only dimensions along which a user
may want to choose different backend parameters. A user may want to submit some
CPU-based tasks to a large-memory queue, and others to a many-core queue based
on which machine topology better supports their needs, but there is not
currently a clear way for WDL implementations to present this choice in
configuration.

A brute-force solution is to expose backend configuration parameters through
implementation-defined hints (preferably registered through a mechanism like a
[hint registry](https://github.com/stjude-rust-labs/rfcs/pull/5)), and then
[override those
hints](https://github.com/openwdl/wdl/blob/wdl-1.2/SPEC.md#specifying--overriding-requirements-and-hints)
for each relevant task in a workflow execution graph. This proposal seeks to
ease the burden on users by introducing a layer of indirection between a set of
desired configuration parameters and the set of tasks to which those parameters
are applied.

## Proposed solution

A new optional hint, `class` of type `String` or `Array[String]`, which contains
zero or more free-form identifiers, should be available for arbitrary
implementation-defined behavior. In the case of Sprocket, the `sprocket.toml`
configuration could contain sections that match classes and override backend
configurations for corresponding tasks. For example, consider the task:

```wdl
version 1.2

task hungry_4_memory {
  input {
    File foo
  }

  command <<<
  wc -l ~{foo}
  >>>

  output {
    Int num_lines = read_int(stdout())
  }

  requirements {
    container: "ubuntu:latest"
  }

  hints {
    max_memory: "2 TiB"
    class: ["large_mem", "xlarge"]
  }
}
```

Rather than running all tasks in the entire workflow in a queue that supports a
2 TiB job, or specifically overriding each call to `hungry_4_memory` with a
different queue parameter, a Sprocket LSF configuration would look like this:

```toml
[[backends]]

# For tasks with a "large_mem" class, run in the "large" queue
[large]
type = "lsf_apptainer"
if_class = "large.*"
default_lsf_queue.name = "large"
default_lsf_queue.max_cpu_per_task = 128
default_lsf_queue.max_memory_per_task = "4 TiB"

# For tasks where no class selectors match, run in the "short" queue
[default]
type = "lsf_apptainer"
default_lsf_queue.name = "short"
default_lsf_queue.max_cpu_per_task = 4
default_lsf_queue.max_memory_per_task = "16 GiB"
```

When picking which backend to use to execute a particular task, Sprocket would
attempt to match any `class` hints against the regular expression defined in the
optional `if_class` backend configuration field. If there's a match, the
associated backend would be used for that task, with the default backend being
used for any tasks which fall through without matching any `if_class` fields.

## Drawbacks

A user would potentially still have to modify the WDL with `class` hints that
suit their execution environment, though the ability to include multiple classes
and match over them with regular expressions would hopefully lead to classes
being somewhat reusable between engines. If modifying WDL is not feasible, the
hint override mechanism can still be used on a task-by-task basis, and the
ability to override an entire set of flags with a single `class` hint would
still be nicer than making users repeat an entire configuration for each task in
the execution graph.

There is the potential for a fair amount of repetition within the
`sprocket.toml` that would be particularly annoying if only a single field, such
as queue name, needs to change between different classes. However, this seems
like a worthwhile tradeoff to avoid the complexity that arises when trying to
combine blocks of configuration, particularly if a more-specialized backend
configuration requires unsetting an option that is present in a default backend.

## Alternatives

The
[miniwdl-slurm](https://github.com/miniwdl-ext/miniwdl-slurm/blob/develop/README.rst)
plugin adds a small expression language to the `.cfg` file, so users can
directly program predicates to distinguish when different sets of configuration
options should be applied:

```cfg
[slurm]
# extra arguments passed to the sbatch command (optional).
dynamic_partition = [
    {
      "memory__ge": 20000000000,
      "cpu__gt": 30,
      "time_minutes__gt": 7200,
      "args": "--partition highmem --comment highmem-long"
    },
    {
      "memory__lt": 2000000000,
      "time_minutes__lt": 60,
      "args": "--comment short"
    },
    {
      "args": "--comment default"
    }
  ]
```


This allows complex hierarchies of configurations without any modification to
the WDL sources. However, the syntax is ... the best it probably can be given
the constraints, and the desired logic must be expressible as a series of
conjunctions.

The properties available for inspection by these predicates are fixed by the
implementation: `cpu`, `time_minutes`, etc must all be hard-coded. It's not hard
to imagine a scenario where a user wishes to discriminate based on some other
property of the task and ends up with a similar problem to the status quo.

Finally, while it's good to have a solution that doesn't require changing WDL,
it also limits the ability for the benefits of the task classification to be
shared with other users.
