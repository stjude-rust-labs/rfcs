# Hint registry for OpenWDL

The `hints` section in WDL 1.2+ explicitly invites implementations to define and use hints in concert with other implementations (see the final bullet point in [Conventions and Best Practices](https://github.com/openwdl/wdl/blob/wdl-1.2/SPEC.md#conventions-and-best-practices):

> Users tend to look for similar behavior between different execution engines. It is strongly encouraged that implementers of execution engines agree on common names and accepted values for hints that describe common usage patterns.

This issue proposes a means for these hints to be centrally registered to facilitate this sharing, but with a lower bar for entry than additions to the set of [Reserved Hints](https://github.com/openwdl/wdl/blob/wdl-1.2/SPEC.md#reserved-task-hints).

The registry would be a table of hint names, accepted types, task/workflow scope, aliases, default values, short descriptions, and links to additional information. To slightly modify the documentation of the [HTTP Field Names Registry](https://github.com/protocol-registries/http-fields):

> Registration is intended to:
>
> 1. Avoid conflicting uses of a hint
> 2. Allow the documentation associated with a hint to be discovered
> 3. Offer guidance regarding a hint's definition during the process of registration

## Initial registry contents

The initial registry would contain the reserved hints specified in the WDL specification. As CSV:

```csv
Name,Accepted Types,Task,Workflow,Aliases,Default Value,Short Description,Reference
max_cpu,"Int, Float",Yes,No,maxCpu,,A hint to the execution engine that the task expects to use no more than the specified number of CPUs.,https://github.com/openwdl/wdl/blob/wdl-1.2/SPEC.md#max_cpu
disks,"String, Map[String, String]",Yes,No,,,A hint to the execution engine to mount disks with specific attributes.,https://github.com/openwdl/wdl/blob/wdl-1.2/SPEC.md#-disks
gpu,"Int, String",Yes,No,,,A hint to the execution engine to provision hardware accelerators with specific attributes.,https://github.com/openwdl/wdl/blob/wdl-1.2/SPEC.md#-gpu-and--fpga
fpga,"Int, String",Yes,No,,,A hint to the execution engine to provision hardware accelerators with specific attributes.,https://github.com/openwdl/wdl/blob/wdl-1.2/SPEC.md#-gpu-and--fpga
short_task,Boolean,Yes,No,,false,A hint to the execution engine about the expected duration of this task.,https://github.com/openwdl/wdl/blob/wdl-1.2/SPEC.md#short_task
localization_optional,Boolean,Yes,No,localizationOptional,false,A hint to the execution engine about whether the File inputs for this task need to be localized prior to executing the task.,https://github.com/openwdl/wdl/blob/wdl-1.2/SPEC.md#localization_optional
inputs,input,Yes,No,,,Provides input-specific hints. Each key must refer to a parameter defined in the task's input section.,https://github.com/openwdl/wdl/blob/wdl-1.2/SPEC.md#inputs
allow_nested_inputs,Boolean,No,Yes,allowNestedInputs,,Let the user set the value of some call inputs at runtime.,https://github.com/openwdl/wdl/blob/wdl-1.2/SPEC.md#allow_nested_inputs
```

CSV or another machine-readable format would be useful for implementors to add a baseline level of validation for registered hints even if they do not fully support them. GitHub doesn't render CSV nicely within Markdown, though, so here's a human-readable table version:

|Name                 |Accepted Types             |Task|Workflow|Aliases             |Default Value|Short Description                                                                                                           |Reference                                                                |
|---------------------|---------------------------|----|--------|--------------------|-------------|----------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------|
|max_cpu              |Int, Float                 |Yes |No      |maxCpu              |             |A hint to the execution engine that the task expects to use no more than the specified number of CPUs.                      |https://github.com/openwdl/wdl/blob/wdl-1.2/SPEC.md#max_cpu              |
|disks                |String, Map[String, String]|Yes |No      |                    |             |A hint to the execution engine to mount disks with specific attributes.                                                     |https://github.com/openwdl/wdl/blob/wdl-1.2/SPEC.md#-disks               |
|gpu                  |Int, String                |Yes |No      |                    |             |A hint to the execution engine to provision hardware accelerators with specific attributes.                                 |https://github.com/openwdl/wdl/blob/wdl-1.2/SPEC.md#-gpu-and--fpga       |
|fpga                 |Int, String                |Yes |No      |                    |             |A hint to the execution engine to provision hardware accelerators with specific attributes.                                 |https://github.com/openwdl/wdl/blob/wdl-1.2/SPEC.md#-gpu-and--fpga       |
|short_task           |Boolean                    |Yes |No      |                    |false        |A hint to the execution engine about the expected duration of this task.                                                    |https://github.com/openwdl/wdl/blob/wdl-1.2/SPEC.md#short_task           |
|localization_optional|Boolean                    |Yes |No      |localizationOptional|false        |A hint to the execution engine about whether the File inputs for this task need to be localized prior to executing the task.|https://github.com/openwdl/wdl/blob/wdl-1.2/SPEC.md#localization_optional|
|inputs               |input                      |Yes |No      |                    |             |Provides input-specific hints. Each key must refer to a parameter defined in the task's input section.                      |https://github.com/openwdl/wdl/blob/wdl-1.2/SPEC.md#inputs               |
|allow_nested_inputs  |Boolean                    |No  |Yes     |allowNestedInputs   |             |Let the user set the value of some call inputs at runtime.                                                                  |https://github.com/openwdl/wdl/blob/wdl-1.2/SPEC.md#allow_nested_inputs  |

## Hints registry repository

A new GitHub repository should be created under the `openwdl` organization for the purpose of maintaining the hint registry (if the concept of a registry would be useful for other WDL constructs such as `meta` or `parameter_meta`, perhaps it shouldn't be limited to hints). The README and pull request template of this repository would spell out the process of registering a new hint and provide examples of extending the CSV. Once a PR adding a hint is accepted, that's largely the end of the process. It's now included by reference in the specification.

## Extension of the spec

The registry repository should be described by and linked from the relevant portions of the WDL spec to formalize it as the shared source of truth for widely-recognized hints.
