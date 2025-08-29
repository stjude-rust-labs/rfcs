- Feature Name: sprocket-test
- Start Date: 2025-08

# Summary
[summary]: #summary

The Sprocket test framework enables WDL authors to easily and comprehensively validate their WDL tasks and workflows by defining lightweight unit tests that can run in CI environments. This framework is intended to be intuitive and concise.

This framework can maximize test depth without adding boilerplate by allowing users to define "test matrices", where each WDL task or workflow is run with permutations of the provided inputs.

# Motivation
[motivation]: #motivation

Comprehensive unit testing is a key component of modern software development. Any serious project should endeavor to have a suite of tests that ensure code correctness. These tests should be lightweight enough to run during continuous integration on every set of committed changes. (This RFC is primarily focused on CI unit testing, but enabling larger scale "end-to-end" testing is something that should be kept in mind during this design process. That said, I'm of the opinion these are separate enough use cases that they can be approached with differing APIs, and thus decisions made here should not impact any future E2E testing API too heavily.) 

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The Sprocket test framework is primarily specified in TOML, which is expected to be within a `tests/` directory at the root of the WDL workspace. `sprocket test` does not require any special syntax or modification of actual WDL files, and any spec-compliant WDL workspace can create a `tests` directory respected by Sprocket.

The `tests/` directory is expected to mirror the WDL workspace, where it should have the same path structure, but with `.wdl` file extensions replaced by `.toml` extensions. The TOML files contain tests for tasks and workflows defined at their respective WDL counterpart in the main workspace. e.g. a WDL file located at `data_structures/flag_filter.wdl` would have accompanying tests defined in `tests/data_structures/flag_filter.toml`. Following this structure frees the TOML from having to contain any information about _where_ to find the entrypoints of each test. All the test entrypoints (tasks or workflows) in `tests/data_structures/flag_filter.toml` are expected to be defined in the WDL located at `data_structures/flag_filter.wdl`.

Any task or worklow defined in `data_structures/flag_filter.wdl` can have any number of tests associated with it in `tests/data_structures/flag_filter.toml`. To write a set of tests for a task `validate_string_is_12bit_int` in `flag_filter.wdl`, the user will define an array of tables in their TOML with the key `[[validate_string_is_12bit_int]]`. Multiple tests can be written for the `validate_string_is_12bit_int` task by repeating the `[[validate_string_is_12bit_int]]` header. The `validate_flag_filter` workflow can be tested the same way, by defining a TOML array of tables headered with `[[validate_flag_filter]]`. Under each of these headers will be a TOML table with all the required information for a single test.

An example TOML for specifying a suite of tests for [the `flag_filter.wdl` document in the `workflows` repo](https://github.com/stjudecloud/workflows/blob/main/data_structures/flag_filter.wdl) would look like:

```toml
[[validate_string_is_12bit_int]]
name = "decimal passes" # each test must have a unique name
[validate_string_is_12bit_int.inputs] # table of inputs
number = "5"
# without any tests explicitely configured, Sprocket will consider the task executing with a 0 exit code to be a "pass" and any non-zero exit code as a "fail"

[[validate_string_is_12bit_int]]
name = "hexadecimal passes"
[validate_string_is_12bit_int.inputs]
number = "0x900"
[validate_string_is_12bit_int.tests] # table of test conditions
exit_code = 0 # redundant and not needed explicitly
stdout.contains = "Input number (0x900) is valid" # builtin test for checking STDOUT logs

[[validate_string_is_12bit_int]]
name = "too big hexadecimal fails"
[validate_string_is_12bit_int.inputs]
number = "0x1000"
[validate_string_is_12bit_int.tests]
exit_code = 42 # the task should fail for this test
stderr.contains = "Input number (0x1000) is invalid" # similar to the stdout test

[[validate_string_is_12bit_int]]
name = "too big decimal fails"
[validate_string_is_12bit_int.inputs]
number = "4096"
[validate_string_is_12bit_int.tests]
exit_code = 42
stderr.contains = [
    "Input number (4096) interpreted as decimal",
    "But number must be less than 4096!",
] # a test which is a bit more complicated

[[validate_flag_filter]] # a workflow test
name = "Valid FlagFilter passes"
[validate_flag_filter.inputs.flags]
include_if_all = "3" # decimal
exclude_if_any = "0xF04" # hexadecimal
include_if_any = "03" # octal
exclude_if_all = "4095" # decimal

[[validate_flag_filter]]
name = "Invalid FlagFilter fails"
[validate_flag_filter.inputs.flags]
include_if_all = "" # empty string
exclude_if_any = "this is not a number"
include_if_any = "000000000011" # binary interpreted as octal. Too many digits for octal
exclude_if_all = "4095" # this is fine
[validate_flag_filter.tests]
should_fail = true
```

Hopefully, everything in the above TOML is easily enough grokked that I won't spend time going through the specifics in much detail. The `flag_filter.wdl` WDL document contains a task and a workflow, both with minimal inputs and no outputs, making the tests fairly straightforward. One of Sprocket's guiding principles is to only introduce complexity where it's warranted, and I hope that this example demonstrates a case where complexity is _not_ warranted. Next, we will be discussing features intended for allowing test cases that are more complex, but the end API exposed to users (the focus of this document) still aims to maintain simplicity and inuitiveness. 

## Test Data

Most WDL tasks and workflows have `File` type inputs and outputs, so there should be an easy way to incorporate test files into the framework. This can be accomplished with a `tests/fixtures/` directory that can be referred to from any TOML test in the `tests` directory. If the string `$FIXTURES` is found within a TOML string value within the `inputs` table, the correct path to the `fixtures` directory will be dynamically inserted at test run time. This avoids having to track relative paths from TOML that may be arbitrarily nested in relation to test data. For example, let's assume there are `test.bam`, `test.bam.bai`, and `referece.fa.gz` files located within the `tests/fixtures/` directory; the following TOML `inputs` table could be used regardless of where that acutal `.toml` file resides within the `tests/` directory:

```toml
bam = "$FIXTURES/test.bam"
bam_index = "$FIXTURES/test.bam.bai"
reference_fasta = "$FIXTURES/reference.fa.gz"
```

## Builtin Test Conditions

Sprocket will support a variety of common test conditions. In this document so far, we've seen a few of the most straightforward conditions already in the `tests` table of the earlier TOML example (`exit_code`, `stdout.contains`, `stderr.contains` and `should_fail`). For the initial release of `sprocket test`, these test builtins will probably remain as a rather small and tailored set, but the implementation should make extending this set in subsequent releases simple and non-breaking.

Some tests that might exist at initial release:

- `exit_code = <int>` (should an array of ints be supported?)
- `should_fail = <bool>`: only available for workflow tests! Task tests should instead specify an `exit_code`
- `stdout`: will be a TOML table with a sub-tests related to checking a tasks STDOUT log (_not available_ for workflow tests)
    - `contains = <string | array of strings>`: strings are REs
    - `not_contains = <string | array of strings>`: strings are REs
- `stderr`: functionally equivalent to the `stdout` tests, but runs on the STDERR log instead
- `outputs`: a TOML table populated with task or workflow output indentifiers. The specific tests available will depend on the WDL type of the specified output
    - `<WDL Boolean> = <true|false>`
    - `<WDL Int> = <TOML int>`
    - `<WDL Float> = <TOML float>`
    - `<WDL String>`
        - `contains = <string | array of strings>`: strings are REs
        - `not_contains = <string | array of strings>`: strings are REs
        - `equals = <string>` an exact match RE
    - `<WDL File>`
        - `name = <string>`: glob pattern that should match
        - `hash = <string>`: md5 (or blake3?) that the file should have
        - `contains = <string | array of strings>`: REs to expect (implies the file is ASCII and will fail if in another encoding)
        - `not_contains = <string | array of strings>`: inverse of `contains`

The above is probably (about) sufficient for an initial release. Thoughts about future tests that could exist will be discussed further down this document.

## Test Matrices

Often, it makes sense to validate that a variety of inputs result in the same test result. While the TOML definitions shared so far are relatively concise, repeating the same test conditions for many different inputs can get repetitive and the act of writing redundant boilerplate can discourage testing best practices. Sprocket offers a "shortcut" for avoiding this boilerplate, by defining test matrices. These test matrices can be a way to reach comprehensive test depth with minimal boilerplate test definitions. A test matrix is created by defining a `matrix` TOML array of tables for a set of test inputs. Each permutation of the "input vectors" will be run, which can be leveraged to test many conditions with a single test definition. An example for a `bam_to_fastq` task might look like:

```toml
[[bam_to_fastq]]
name = "Kitchen Sink"
[[bam_to_fastq.matrix]]
bam = [
    "$FIXTURES/test1.bam",
    "$FIXTURES/test2.bam",
    "$FIXTURES/test3.bam",
]
bam_index = [
    "$FIXTURES/test1.bam.bai",
    "$FIXTURES/test2.bam.bai",
    "$FIXTURES/test3.bam.bai",
]
[[bam_to_fastq.matrix]]
bitwise_filter = [
    { include_if_all = "0x0", exclude_if_any = "0x900", include_if_any = "0x0", exclude_if_all = "0x0" },
    { include_if_all = "00", exclude_if_any = "0x904", include_if_any = "3", exclude_if_all = "0" },
]
[[bam_to_fastq.matrix]]
paired_end = [true, false]
[[bam_to_fastq.matrix]]
retain_collated_bam = [true, false]
[[bam_to_fastq.matrix]]
append_read_number = [true, false]
[[bam_to_fastq.matrix]]
output_singletons = [true, false]
[bam_to_fastq.inputs]
prefix = "kitchen_sink_test" # the `prefix` input will be shared by _all_ permutations of the test matrix
# this test is to ensure all the options (and combinations thereof) are valid
# so no tests beyond a `0` exit code are needed here
```

This is perhaps an extreme test case, but it was contrived as a stress test of sorts for the matrix design. In ~30 lines of TOML, this defines `3*2*2*2*2*2 = 96` different permutations of the `bam_to_fastq` task that should be executed by Sprocket. This specific case may be too intense to run in a CI environment, but should demonstrate the power of test matrices in aiding comprehensive testing without undue boilerplate.

(Notably, the actual `bam_to_fastq` task in `samtools.wdl` ([here](https://github.com/stjudecloud/workflows/blob/main/tools/samtools.wdl#L763)) does not have a `bam_index` input, but that was added to this example for illustrative purposes)

REVIEWERS: I can write more examples of "real" TOML test files, as presumably we will be switching the `workflows` repo to this framework, in which case any tests written as examples here can hopefully just be re-used with minimal modification for the production tests we want. So don't be afraid to ask for more examples! I just didn't want to overload this document ;D

## Configuration

Both of the expected paths, `tests/` and `fixtures/` within `tests/`, can be overriden via config. `tests/` conflicts with `pytest-workflow`, so users may want to rename the default Sprocket test directory to something like `sprocket-tests/`. It is unlikely that anyone will be writing WDL that needs to be tested at a `fixtures/` path, but this should also be configurable to avoid that potential clash.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

REVIEWERS: is this section needed?

-----

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

Q: Why should we *not* do this?

A: _none!_ This is a great idea!

To be serious, `pytest-workflow` seems to be the best test framework for WDL that I've been able to find as a WDL author, and as a user of that framework, I think WDL could use something more tailored. I will elaborate further in [Prior Art](#prior-art).

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

REVIEWERS: I've thought through quite a wide variety of implementations that have not made it into writing, and I'm not sure how valuable my musings on alternatives I _didn't like_ are. I can expand on this section if it would be informative, but I am leaving the below questions unanswered for now.

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

This document has been largely informed by my experience as a WDL author and maintainer of the [stjudecloud/workflows](https://github.com/stjudecloud/workflows) repository. The CI of that repo uses `pytest-workflow`.

## `pytest-workflow`

[`pytest-workflow`](https://github.com/lumc/pytest-workflow/) has been a great stop-gap tool for us. It is a generalized test framework not specific to WDL, which is ultimately what makes it unwieldly for our use cases. The WDL community should have a better solution than a generic test framework.

That said, if you are familiar with pytest-workflow, you will likely see some similarities between it and my proposal. I've leaned on the existing designs used in pytest-workflow, and made them more specific and ergonomic for WDL.

REVIEWERS: I can elaborate further if asked

## [wdl-ci](https://github.com/DNAstack/wdl-ci)

I found this tool while looking for existing frameworks when starting this document; which is to say it's new to me and I have not tried running it, but it has some interesting capabilities.

This is _not_ a unit testing framework, and looks geared towards system testing or end-to-end testing, or whatever term you want to use for ensuring _consistency and reproducibility_ while developing a workflow. Definitely something worth revisiting when we circle back to that use case, but at the moment this is just designed for a different use than my proposal.

## [pytest-wdl](https://github.com/EliLillyCo/pytest-wdl)

Last update was 4 years ago, which we on the `workflows` repo considered a deal breaker when we were iniitally shopping for CI testing. It seems very similar to pytest-workflow, at least on the surface, (of course they are both plugins to the popular pytest framework) but admittedly I have not dove very deep into this project.

## [CZ ID repo](https://github.com/chanzuckerberg/czid-workflows/tree/main?tab=readme-ov-file#cicd)

This is just a WDL repo, not a full test framework, but they do have a bespoke CI/CD set up that I reviewed. Uses miniwdl under the hood and seems worth mentioning, but not a generalizable approach.

# Future possibilities
[future-possibilities]: #future-possibilities

I think there's a lot of room for growth in the builtin test conditions. This document just has what I think are about appropriate for an initial release (i.e. are relatively easy to implement), but that shouldn't be the end of the builtin tests. I can imagine us writing bioinformatics specific tests using the `noodles` library for testing things like "is this output file a valid BAM?", "is this BAM coordinate sorted?", "is the output BAI file correctly matched to the output BAM?", and many many more such tests.

Another possibility we may even want to expose in an initial relase (so move from "Future possibilities" to a more prominent section of this RFC) would be custom tests. There are a couple of directions I've considered for this, and I think the most logical jumping off point would be to let users write shell scripts that are invoked with ENV vars set with the outputs of a task or workflow, and let them do whatever the heck tests they want to write in Bash and just ensure the exit code of the script is `0`. This seems to me the best ROI from an implementation standpoint. It would be pretty easy to set up and shell out, and would allow users to write arbitrarily complicated tests using whatever tools they want.

As stated in the "motivation" section, this proposal is ignoring end-to-end (or E2E) tests and is really just focused on enabling unit testing for CI purposes. Perhaps some of this could be re-used for an E2E API, but I have largely ignored that aspect. (Also I have lots of thoughts about what that might look like, but for brevity will not elaborate further.)

