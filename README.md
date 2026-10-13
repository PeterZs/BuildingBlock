# BuildingBlock

**[SIGGRAPH 2025] [BuildingBlock: A Hybrid Approach for Structured Building Generation](https://arxiv.org/pdf/2505.04051)**

![BuildingBlock pipeline](https://github.com/user-attachments/assets/1fa83ce6-4152-4277-811f-79849562cabc)
![BuildingBlock results](https://github.com/user-attachments/assets/2fbe3d2c-89cf-4583-9e95-04eb447886a1)

## Release Scope

This repository provides the **Stage 1 component-layout generation** code and data preparation workflow.

**The original Stage 2 implementation involves proprietary technology and trade secrets and cannot be open-sourced.**

For a lightweight community alternative, you can use a frontier LLM coding agent to create a reusable skill that assigns attributes to layout components and assembles a 3D building procedurally with Python and Blender. We provide a self-contained generation prompt below.

## Installation

### 1. Install Docker and Pull the Image

Install Docker by following the instructions on [Docker's official website](https://www.docker.com/get-started), then pull the project image:

```bash
docker pull dreaminghuang/building_block:0.1
```

### 2. Run the Docker Container

Mount your local project directory and start an interactive session:

```bash
docker run -it --gpus all \
  -v "your_path_of_building_block:/building_block" \
  -w /building_block \
  dreaminghuang/building_block:0.1 /bin/bash
```

Replace `your_path_of_building_block` with the absolute path to your local project directory. Ensure GPU drivers and Docker GPU support are configured correctly.

### 3. Initialize the Environment

From the project root inside the container, run:

```bash
bash initialization.sh
```

## Dataset Preparation

### 1. Download and Extract the Dataset

Download the dataset from [Hugging Face](https://huggingface.co/datasets/dreaming-huang/buildingblock/blob/main/building_block_data_opensource_only_layout_and_cond.zip).

Extract the archive and locate the `BoxCenterSizeLabel_all/` directory:

```bash
unzip building_block_data_opensource_only_layout_and_cond.zip
```

### 2. Process the Dataset

Run the following commands in sequence from the project root:

```bash
# Apply rotation-based data augmentation.
python 1-json_rotate_augment.py

# Normalize the Unreal Engine JSON layout data.
python 2-normUeJson.py

# Convert JSON layouts into NumPy box representations.
python 3-json2boxnp.py

# Copy the dataset statistics into the processed-data directory.
cp dataset_stats.txt ./BoxCenterSizeLabelNp/
```

Processing time depends on the dataset size and available resources. Check the script output and logs for errors before proceeding.

## Running Stage 1

Navigate to the scripts directory:

```bash
cd scripts
```

Review `command.sh` for the available training and evaluation commands. For example:

```bash
python train_diffusion_building_DDP.py \
  ../config/text/diffusion_building_DIT.yaml \
  uncond \
  --experiment_tag uncond \
  --n_processes 0 \
  --with_swanlab_logger
```

Check the paths and settings against your local checkout before running. Refer to the configuration files and individual scripts for model architecture and training parameters.

## Stage 2: Generate a Community Skill with GPT-6 Astra

The original Stage 2 cannot be released because it involves proprietary technology and trade secrets. As an alternative, the prompt below asks GPT-6 Astra to build a reusable, local **layout-to-3D skill** around this workflow:

**Component layout → attribute assignment → procedural geometry → Blender assembly and export**

### How to Use

1. Open GPT-6 Astra in a coding-agent environment that can create files and run local Python/Blender commands.
2. Provide a **normalized component-layout JSON** and, where available, the public repository or relevant preprocessing scripts. Follow the paper for component categories and use the public code to resolve their exact encoding. If the layout is stored as NumPy arrays, provide the corresponding schema or conversion code rather than asking the model to guess.
3. Copy the prompt below. An architectural style is optional. A local Blender installation is needed to execute and validate the generated skill; it is separate from the Stage 1 training setup.
4. Review the generated code and its validation results before using it on additional layouts. Further iteration may be needed for unsupported categories or visual artifacts.

### Copy-and-Paste Prompt

```text
Create a complete, reusable agent skill named `buildingblock-layout-to-3d`.

Goal
Given a BuildingBlock component layout, assign suitable attributes to each
component, then use Python and Blender to procedurally construct and assemble
an editable 3D building. Generate the skill itself, including executable scripts
and usage instructions, rather than only describing a solution or generating
one building.

Reference and scope
- Read BuildingBlock: A Hybrid Approach for Structured Building Generation:
  https://arxiv.org/pdf/2505.04051
- Follow the paper's component taxonomy. Resolve exact category tokens, IDs,
  aliases, and coordinate conventions from the public preprocessing code and
  the supplied layout. Do not invent a separate label system. If a necessary
  mapping is unavailable, ask for it instead of guessing.
- Implement an independent community alternative. Do not claim to reproduce
  the proprietary Stage 2, recover its assets, or match the paper's results.
- Use original procedural geometry and materials. Do not require private
  assets, private code, or a pre-existing skill package.

1. Read and preserve the layout
Validate component identifiers, categories, box centers, box sizes, coordinate
axes, and normalization. Distinguish full extents from half extents. Preserve
all source components, categories, positions, and sizes; do not redesign the
layout. Do not reapply rotations or mirrors already present in the input.
Permit only an explicitly recorded global translation and uniform display
scale. Treat physical dimensions as an assumption unless supplied by the user.

2. Assign component attributes
Write exactly one record per source component to `choices.json`. Based on its
category, box proportions, spatial relationships, and the requested style,
choose a semantic role, procedural geometry type, materials, and appropriate
detail parameters. Include a short rationale. Keep repeated and adjacent
components stylistically consistent. Infer facade orientation and attachment
to host components when needed, but mark these as inferred rather than source
data. Use a coherent default style when none is requested.

3. Build and assemble with Python and Blender
Separate agent-selected attributes from deterministic geometry generation.
Provide a scene-preparation script and a Blender assembly script. Use a
category-to-builder registry and report coverage for every category present
in the input. Implement the necessary builders; if a category remains
unsupported, stop with a clear explanation rather than dropping components
or silently replacing them with generic boxes.
Place geometry at the supplied boxes. Keep details within the intended
component envelopes and scale details consistently. Ensure facade elements
face outward and doors/windows are not buried in solid walls. Record any
wall-opening or attachment operations separately from immutable source boxes.
Keep each component independently identifiable and editable in Blender.

4. Validate and deliver
Preserve the input and generate a manifest linking each source component to
its selected attributes and constructed objects, including the global
transform and any inferred relationships. Check component coverage, category
consistency, finite transforms, positive sizes, placement errors, and geometry
bounds. Report actual measured errors and tolerances; do not equate keeping
geometry inside a box with exactly matching its full bounds.
Export an editable `.blend`, a portable `.glb`, multi-view PNG previews, the
attribute choices, the manifest, and a machine-readable validation report.
Inspect the previews for hidden openings, incorrect orientation, missing parts,
and obvious geometry artifacts. Fix issues and rerun where possible. Keep
presentation-only ground, lights, and cameras out of the building-only GLB.
State any material/export differences and unresolved limitations.

5. Package the reusable skill
Include `SKILL.md` with name/description metadata, concise input/output and
workflow documentation, `scripts/`, and a reference describing the layout and
attribute contracts. Accept input/output paths at invocation time; do not
hard-code personal paths, component counts, or private sample data.
Document the Python and Blender versions used. Run Blender in a fresh
background process, not in the user's existing scene. Keep writes inside a
specified output directory and avoid overwriting existing work without
confirmation. Do not automatically download assets or install global packages.
Test first on a small synthetic layout, then on the supplied layout if one is
available. Distinguish tests actually run from tests that could not be run.
If Blender or execution access is unavailable, still provide the complete
skill files and exact commands, but clearly mark runtime validation as pending.

Return the skill directory structure, complete file contents or downloadable
files, usage commands, test results, and limitations. Do not leave required
scripts as pseudocode or TODOs. A successful result is a reusable procedural
layout-to-3D skill, not an exact reconstruction of the original building.
```

The generated skill should work with new layouts through user-supplied paths rather than depending on the sample used during development. Support for additional categories or architectural styles may require additional procedural builders.

## Project Structure

- `1-json_rotate_augment.py`: Layout rotation augmentation.
- `2-normUeJson.py`: Unreal Engine JSON layout normalization.
- `3-json2boxnp.py`: Conversion from JSON layouts to NumPy box representations.
- `scripts/`: Model training and evaluation scripts, including `command.sh`.
- `config/`: Model configuration files, as referenced by the training example above; verify the directory name in your checkout.
- `BoxCenterSizeLabel_all/`: Extracted source layout dataset.
- `BoxCenterSizeLabelNp/`: Processed NumPy layout data and dataset statistics.

The community skill is generated separately by following the prompt above; its source files are not included in this repository.
