# Bioimage Analysis Provenance Convention Metadata

- **UUID**: 630194dc-6c04-43b2-957a-a80ebb901c42
- **Name**: bioimage_analysis:
- **Schema URL**: "https://raw.githubusercontent.com/BioinfoTongLI/bioimage-analysis-spec/refs/tags/v0.1.0/schema.json"
- **Spec URL**: "https://github.com/BioinfoTongLI/bioimage-analysis-spec/blob/v0.1.0/README.md"
- **Scope**: Array, Group
- **Extension Maturity Classification**: Proposal
- **Owner**: @BioinfoTongLI

## Description

This convention records **which analysis steps produced or modified a Zarr node**: the software and version that ran, the parameters it ran with, the data it read and wrote, the container and workflow it ran in, and when. All properties use the `bioimage_analysis:` namespace prefix and are placed at the root `attributes` level, following the [Zarr Conventions Specification](https://github.com/zarr-conventions/zarr-conventions-spec).

Version 0.1 is deliberately small. It contains only the provenance fields that existing bioimage tools and data producers already record in common (see [Survey of existing practice](#survey-of-existing-practice)). Anything more specific can be added by other conventions or tools, because every object defined here is extensible.

- Examples:
  - [Minimal: only the required fields](examples/minimal_example.json)
  - [Format conversion, mapped from Janelia TensorSwitch](examples/tensorswitch_conversion_example.json)
  - [Format conversion, mapped from the BioImage Archive](examples/bia_conversion_example.json)
  - [Four-step pipeline, mapped from an Allen Institute `processing.json`](examples/allen_multistep_example.json)
  - [Segmentation step of a Fractal / Nextflow pipeline](examples/fractal_nextflow_label_example.json)
  - [The same, as a Zarr v2 `.zattrs` next to OME-NGFF 0.4 metadata](examples/zarr_v2_zattrs_example.json)

## Motivation

- **Provenance travels with the data**: Image data are copied, converted and re-hosted. Provenance stored in a side file or a database is easily lost; provenance in the node's own attributes is not.
- **One place to look**: Today, each tool writes its own key (`_software`, `tensorswitch`, `mia`, `additional_metadata`, `processing.json`, ...). Readers must know every tool to find out what happened to an image.
- **Multi-step analysis**: A label image or a table is usually the result of several steps (conversion, preprocessing, segmentation, measurement). An ordered list of steps captures that history.
- **Use case alignment**: Answering "which software, version and parameters made this?" when writing a paper, reproducing a result, or deciding whether two datasets were processed the same way.

## Convention Registration

The convention must be registered in `zarr_conventions`:

```json
{
  "zarr_conventions": [
    {
      "uuid": "630194dc-6c04-43b2-957a-a80ebb901c42",
      "schema_url": "https://raw.githubusercontent.com/BioinfoTongLI/bioimage-analysis-spec/refs/tags/v0.1.0/schema.json",
      "spec_url": "https://github.com/BioinfoTongLI/bioimage-analysis-spec/blob/v0.1.0/README.md",
      "name": "bioimage_analysis:",
      "description": "Provenance of the bioimage analysis steps that produced or modified this node"
    }
  ]
}
```

## Applicable To

This convention can be used with these parts of the Zarr hierarchy:

- [x] Group
- [x] Array

## Properties

All properties are placed at the root `attributes` level and use the key-prefixed pattern.

**Convention metadata name**: `bioimage_analysis:`

| Field Name              | Type                           | Description                                                                                                          |
| ----------------------- | ------------------------------ | -------------------------------------------------------------------------------------------------------------------- |
| bioimage_analysis:steps | \[[Step Object](#step-object)] | **REQUIRED**. Analysis steps that created or modified this node, in execution order. Must contain at least one step. |

**Example**:

```json
{
  "attributes": {
    "zarr_conventions": [
      {
        "name": "bioimage_analysis:",
        "uuid": "630194dc-6c04-43b2-957a-a80ebb901c42"
      }
    ],
    "bioimage_analysis:steps": [
      {
        "name": "Threshold Segmentation",
        "software": {
          "name": "fractal-tasks-core",
          "version": "2.1.0",
          "uri": "https://pypi.org/project/fractal-tasks-core/2.1.0"
        },
        "parameters": { "segmentation_method": { "method": "Otsu" } },
        "inputs": [{ "path": "B/03/0", "name": "image" }],
        "container": {
          "image": "quay.io/bioinfotongli/fractal-tasks-core:2.1.0",
          "engine": "docker"
        },
        "started": "2026-11-04T11:55:00.431+01:00",
        "ended": "2026-11-04T11:55:02.438+01:00"
      }
    ]
  }
}
```

### Additional Field Information

#### bioimage_analysis:steps

- **Placement.** The attribute is written on the node that a step created or modified: an image group, a label group, a table group, or an array. A tool that modifies a node SHOULD append its step to the existing list rather than replace it.
- **Order.** Steps are listed in the order they ran. The last step is the most recent one.
- **Upstream steps.** A node MAY repeat the steps that produced its inputs (for example, a label image may also list the conversion of the image it was segmented from). Alternatively, upstream steps can be found by following `inputs` to the input node and reading its own `bioimage_analysis:steps`.
- **Zarr v2.** For Zarr v2 hierarchies (e.g. OME-NGFF 0.4), the attributes object is the content of `.zattrs`, and `bioimage_analysis:steps` sits next to keys such as `multiscales` or `image-label`. See [the v2 example](examples/zarr_v2_zattrs_example.json).

### Step Object

One execution of one analysis step: a conversion, a processing step, a segmentation, a measurement, and so on.

| Field Name  | Type                                               | Description                                                                                 |
| ----------- | -------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| software    | [Software Object](#software-object)                | **REQUIRED**. The software that performed the step.                                         |
| name        | string                                             | What the step did, e.g. `Image conversion`, `Threshold Segmentation`.                       |
| id          | string                                             | Identifier of this execution, e.g. a UUID or a workflow engine task id. Unique in the list. |
| description | string                                             | Free-text notes about the step.                                                             |
| parameters  | object                                             | Parameters the software ran with. Free-form.                                                |
| inputs      | \[[Data Reference Object](#data-reference-object)] | Data the step read.                                                                         |
| outputs     | \[[Data Reference Object](#data-reference-object)] | Data the step wrote, if not only this node.                                                 |
| container   | [Container Object](#container-object)              | Container image the step ran in.                                                            |
| workflow    | [Workflow Object](#workflow-object)                | The workflow (pipeline) the step was part of.                                               |
| started     | string                                             | When the step started: ISO 8601 date-time with time zone.                                   |
| ended       | string                                             | When the step ended: ISO 8601 date-time with time zone.                                     |

#### parameters

Parameters SHOULD be recorded as the software received them, after defaults were applied where the software makes that possible, so that the step can be repeated. Parameters that only point to the data being processed (for example a temporary input path) MAY be left out when the same data is described in `inputs` or `outputs`. Command-line tools MAY record their arguments as a list under a single key (see the [BioImage Archive example](examples/bia_conversion_example.json)).

### Software Object

| Field Name | Type   | Description                                                                          |
| ---------- | ------ | ------------------------------------------------------------------------------------ |
| name       | string | **REQUIRED**. Name of the software, package or tool.                                 |
| version    | string | **REQUIRED**. Released version, e.g. `2.1.0`. Use the commit if there is no release. |
| uri        | string | Where the software can be found, e.g. its repository or package page.                |
| commit     | string | Version-control commit of the software that was run.                                 |

### Workflow Object

| Field Name | Type   | Description                                    |
| ---------- | ------ | ---------------------------------------------- |
| name       | string | **REQUIRED**. Name of the workflow.            |
| version    | string | Version of the workflow.                       |
| uri        | string | Where the workflow source can be found.        |
| commit     | string | Version-control commit of the workflow source. |

### Container Object

| Field Name | Type   | Description                                                  |
| ---------- | ------ | ------------------------------------------------------------ |
| image      | string | **REQUIRED**. Image reference, e.g. `quay.io/org/image:tag`. |
| engine     | string | Container engine, e.g. `docker`, `singularity`, `apptainer`. |
| digest     | string | Content digest of the image, as a [checksum](#checksums).    |

### Data Reference Object

At least one of `path`, `uri` or `id` is **REQUIRED**.

| Field Name | Type   | Description                                                                 |
| ---------- | ------ | --------------------------------------------------------------------------- |
| path       | string | Path of a node relative to the root of this Zarr hierarchy, e.g. `B/03/0`.  |
| uri        | string | URI of data outside this hierarchy.                                         |
| id         | string | Identifier of the data in a catalogue or archive, e.g. a UUID or accession. |
| name       | string | Human-readable name of the data.                                            |
| checksum   | string | Checksum of the data, as a [checksum](#checksums).                          |

### Checksums

Checksums are written as `<algorithm>:<lowercase hex digest>`, for example `sha256:9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08`.

### Extending

Every object above allows additional properties. Tools and other conventions MAY add their own keys, preferably with their own prefix (e.g. `fractal:task_name` inside a step), without breaking validation against this schema.

## Versioning

The version is part of the schema and spec URLs (`v0.1.0`) and follows semantic versioning:

- **Patch** (`0.1.x`): wording and examples only; no change to what validates.
- **Minor** (`0.x.0`): new optional fields; every document valid under an earlier minor version stays valid.
- **Major** (`x.0.0`): incompatible changes, such as new required fields or changed meaning.

While the major version is `0`, minor versions may also contain incompatible changes.

## Survey of existing practice

Version 0.1 contains the concepts recorded by at least 3 of the 5 sources below. `software.name` and `software.version` are required because every source records them.

| Concept                   | [BioImage Archive][bia] | [TensorSwitch][ts] (Janelia) | MIA                  | [Allen `processing.json`][aind]    | [fractal-multistep-nextflow][fmn] | Sources | In v0.1                    |
| ------------------------- | ----------------------- | ---------------------------- | -------------------- | ---------------------------------- | --------------------------------- | ------- | -------------------------- |
| Software name and version | `conversion_function`   | `_software`                  | snippet name, commit | `code.name`, `code.version`        | yes                               | 5       | `software` (**required**)  |
| Parameters                | `conversion_config`     | `tensorswitch`               | `plan` (digest only) | `code.parameters`¹                 | yes                               | 5       | `parameters`               |
| Inputs                    | input UUID              | `lmvd.source_path`           | `derived_from`       | `code.input_data`                  | yes                               | 5       | `inputs`                   |
| Software URL              |                         | `_software.url`              |                      | `code.url`                         | yes                               | 3       | `software.uri`             |
| Commit                    |                         |                              | `harness_commit`     | `code.commit_hash`                 | yes                               | 3       | `software.commit`          |
| Step name or type         | `provenance`            |                              |                      | `process_type`, `name`             | yes                               | 3       | `name`                     |
| Container or environment  |                         |                              | `env_digest`         | `code.container`                   | yes                               | 3       | `container`                |
| Start and end time        |                         |                              | `verification.time`  | `start_date_time`, `end_date_time` | yes                               | 3       | `started`, `ended`         |
| Workflow or pipeline      |                         |                              | harness              | `pipeline_name`, `pipelines`       | yes                               | 3       | `workflow`                 |
| Outputs                   | (the node itself)       | (the node itself)            | `artifact`           | `output_path`                      | yes                               | 3       | `outputs`                  |
| Compute resources, GPU    |                         |                              |                      | `resources`                        | yes                               | 2       | next version               |
| Trained models            |                         |                              |                      | separate `Model` schema            | yes                               | 2       | next version               |
| Licence                   |                         | `license`                    |                      | in `data_description`              |                                   | 2       | out of scope²              |
| People                    |                         |                              |                      | `experimenters`                    |                                   | 1       | out of scope²              |
| Dependency graph          |                         |                              |                      | `dependency_graph`                 |                                   | 1       | `inputs`/`outputs` instead |

¹ In the surveyed Allen data asset, `code.parameters` was empty and the run settings were recorded in `output_parameters`; the [Allen example](examples/allen_multistep_example.json) maps them to `parameters`, `inputs` and `outputs`.
² Describes the dataset rather than the analysis; better covered by dataset-level metadata.

### Candidates for the next version

- **Compute**: CPU, memory, and accelerators (GPU model, count, memory, driver and CUDA versions). Needed to explain results that depend on hardware, especially for deep learning.
- **Models**: trained models used by a step (name, version, URI, framework, checksum), especially weights mounted at run time and not visible in the parameters.
- **Detailed records**: links to fuller provenance documents (e.g. a per-step JSON record, or a Workflow Run RO-Crate) with checksums.

## Known Implementations

### Libraries and Tools

- **[fractal-multistep-nextflow](https://github.com/BioImageTools/fractal-multistep-nextflow)** - Nextflow pipeline chaining [fractal-tasks-core](https://github.com/fractal-analytics-platform/fractal-tasks-core) tasks on OME-Zarr; writes detailed per-step provenance records and is the source of the Fractal examples.
  - Language: Nextflow / Python
  - Status: Planned (writes its own provenance JSON today; writing this convention is not yet implemented)
  - Maintainer: @BioinfoTongLI
  - Since: 2026-11

### Datasets Using This Convention

None yet.

### Resources

- [Zarr Conventions Specification](https://github.com/zarr-conventions/zarr-conventions-spec)
- [Allen Institute biodata-schema](https://github.com/AllenInstitute/biodata-schema) (`processing.json`, `Code`, `DataProcess`)

_If you implement or use this convention, please add your implementation to this list by submitting a pull request._

## Acknowledgements

Based on the [Zarr conventions template](https://github.com/zarr-conventions/template), which is based on the [STAC extensions template](https://github.com/stac-extensions/template/blob/main/README.md). The survey draws on provenance metadata written by the BioImage Archive, Janelia (TensorSwitch), the MIA project and the Allen Institute.

[bia]: https://www.ebi.ac.uk/bioimage-archive/
[ts]: https://github.com/JaneliaSciComp/tensorswitch
[aind]: https://github.com/AllenInstitute/biodata-schema
[fmn]: https://github.com/BioImageTools/fractal-multistep-nextflow
