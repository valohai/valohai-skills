---
name: valohai-design-pipelines
description: Analyze ML project structure and design Valohai pipelines to orchestrate multi-step workflows. Use this skill when a user wants to create a Valohai pipeline, connect multiple ML steps into an automated workflow, identify pipeline opportunities in their codebase, design data flow between preprocessing/training/evaluation/inference steps, or add conditional logic and parallel execution to pipelines. Triggers on mentions of pipelines, workflows, DAGs, orchestration, multi-step ML, or connecting Valohai steps.
license: Apache-2.0
metadata:
  author: valohai
  version: "1.0"
compatibility: Requires Python ML projects. Designed for Claude Code or similar AI coding agents.
---

# Design Valohai Pipelines

Analyze ML project structure to identify pipeline opportunities and design multi-step Valohai pipelines that orchestrate preprocessing, training, evaluation, and deployment workflows.

## What Pipelines Do

A Valohai pipeline connects individual steps into an automated workflow:
- **Nodes** = Individual steps (preprocessing, training, evaluation, inference, deployment)
- **Edges** = Data flow between steps (outputs become inputs, metadata becomes parameters)

Valohai automatically executes nodes in the right order, passes data between steps, runs independent nodes in parallel, and tracks complete data lineage.

## Step-by-Step: Analyze the Project

### 1. Identify Pipeline Stages

Examine the ML project to find distinct stages. Common patterns:

**Standard ML Pipeline:**
```
Raw Data → Preprocessing → Training → Evaluation → (Deployment)
```

**With Feature Engineering:**
```
Raw Data → Feature Engineering → Train/Val/Test Split → Training → Evaluation
```

**With Hyperparameter Optimization:**
```
Preprocessing → HPO Sweep (Task) → Select Best → Final Training → Evaluation
```

**Multi-Model Pipeline:**
```
                    ┌→ Train Model A ─┐
Preprocessing ──────┤                  ├→ Ensemble → Evaluation
                    └→ Train Model B ─┘
```

**Full Production Pipeline:**
```
Data Ingestion → Preprocessing → Training → Evaluation → Model Validation → Deployment → Smoke Test
```

### 2. Map Data Dependencies

For each stage, identify:
- **What files does it need?** (input data, models, configs)
- **What files does it produce?** (processed data, trained models, metrics, reports)
- **What metrics does it produce?** (accuracy, loss, F1 that downstream steps might need)
- **What parameters should flow between steps?** (best learning rate, selected features)

### 3. Identify Parallelization Opportunities

Steps that don't depend on each other can run in parallel:
- Training multiple model variants
- Processing different data splits
- Running different evaluation metrics
- A/B training with different hyperparameters

## Step-by-Step: Create the Pipeline YAML

### Design Principle: Every Step Should Run Independently

**IMPORTANT**: Whenever possible, give every input a `default` value so each step can be tested and debugged on its own with `vh execution run step-name --adhoc`. Pipeline edges override these defaults at runtime.

Look for default values in the existing code, README, documentation, config files, or data paths already referenced in the project. Only add a default if you can find a real, meaningful value. Do not invent placeholder URLs or dummy paths - leave the input without a default if no real value is available.

### 1. Ensure Steps Are Defined First

Each pipeline node references an existing step. Define all steps before the pipeline:

```yaml
- step:
    name: preprocess-data
    image: python:3.11
    command:
      - pip install -r requirements.txt
      - python preprocess.py {parameters}
    parameters:
      - name: test_split
        type: float
        default: 0.2
    inputs:
      - name: raw-data
        default: s3://bucket/raw-data/

- step:
    name: train-model
    image: pytorch/pytorch:2.1.0-cuda12.1-cudnn8-runtime
    command:
      - pip install -r requirements.txt
      - python train.py {parameters}
    parameters:
      - name: epochs
        type: integer
        default: 50
      - name: learning_rate
        type: float
        default: 0.001
    inputs:
      - name: train-data
        default: s3://bucket/processed/train/
      - name: val-data
        default: s3://bucket/processed/val/

- step:
    name: evaluate-model
    image: pytorch/pytorch:2.1.0-cuda12.1-cudnn8-runtime
    command:
      - pip install -r requirements.txt
      - python evaluate.py
    inputs:
      - name: model
        default: datum://latest-trained-model
      - name: test-data
        default: s3://bucket/processed/test/
```

### 2. Define the Pipeline

```yaml
- pipeline:
    name: training-pipeline
    nodes:
      - name: preprocess
        type: execution
        step: preprocess-data

      - name: train
        type: execution
        step: train-model
        override:
          inputs:
            - name: train-data
            - name: val-data

      - name: evaluate
        type: execution
        step: evaluate-model
        override:
          inputs:
            - name: model
            - name: test-data

    edges:
      # Preprocessing outputs → training inputs
      - [preprocess.output.*train*, train.input.train-data]
      - [preprocess.output.*val*, evaluate.input.test-data]
      - [preprocess.output.*val*, train.input.val-data]

      # Training output → evaluation input
      - [train.output.model*, evaluate.input.model]
```

### 3. Edge Syntax Reference

Edges define data flow: `[source, target]`

#### Source Formats

| Format | Description | Example |
|--------|-------------|---------|
| `node.output.*` | All outputs from a node | `preprocess.output.*` |
| `node.output.*.csv` | Outputs matching extension | `preprocess.output.*.csv` |
| `node.output.name*` | Outputs starting with name | `train.output.model*` |
| `node.output.*name*` | Outputs containing name | `preprocess.output.*train*` |
| `node.metadata.metric` | A logged metric value | `train.metadata.best_lr` |
| `node.parameter.name` | A parameter value | `train.parameter.learning_rate` |

#### Target Formats

| Format | Description | Example |
|--------|-------------|---------|
| `node.input.name` | An input on the target node | `train.input.train-data` |
| `node.parameter.name` | A parameter on the target node | `finetune.parameter.learning_rate` |
| `deploy.file.endpoint.name` | A file for a deployment endpoint | `deploy.file.predict.model` |

### 4. Override Node Defaults

When a node receives data from an edge, clear its default input so the edge value takes priority:

```yaml
nodes:
  - name: train
    type: execution
    step: train-model
    override:
      inputs:
        - name: train-data    # No default - will come from edge
        - name: val-data       # No default - will come from edge
      parameters:
        - name: epochs
          default: 200         # Override the step's default
```

## Advanced Pipeline Patterns

### Parallel Training

```yaml
- pipeline:
    name: parallel-training
    nodes:
      - name: preprocess
        type: execution
        step: preprocess-data

      - name: train-resnet
        type: execution
        step: train-model
        override:
          parameters:
            - name: model_type
              default: resnet50
          inputs:
            - name: train-data

      - name: train-efficientnet
        type: execution
        step: train-model
        override:
          parameters:
            - name: model_type
              default: efficientnet_b0
          inputs:
            - name: train-data

      - name: select-best
        type: execution
        step: compare-models
        override:
          inputs:
            - name: model-a
            - name: model-b

    edges:
      - [preprocess.output.*, train-resnet.input.train-data]
      - [preprocess.output.*, train-efficientnet.input.train-data]
      - [train-resnet.output.model*, select-best.input.model-a]
      - [train-efficientnet.output.model*, select-best.input.model-b]
```

### Hyperparameter Sweep with Task Node

```yaml
- pipeline:
    name: hpo-pipeline
    nodes:
      - name: preprocess
        type: execution
        step: preprocess-data

      - name: hpo-sweep
        type: task
        step: train-model
        override:
          inputs:
            - name: train-data

      - name: deploy-best
        type: execution
        step: deploy-model

    edges:
      - [preprocess.output.*, hpo-sweep.input.train-data]
      - [hpo-sweep.output.model*, deploy-best.input.model]
```

### Conditional Execution (Actions)

```yaml
- pipeline:
    name: conditional-pipeline
    nodes:
      - name: train
        type: execution
        step: train-model
        actions:
          # Stop pipeline if accuracy is too low
          - when: node-complete
            if: metadata.accuracy < 0.5
            then: stop-pipeline

      - name: evaluate
        type: execution
        step: evaluate-model
        actions:
          # Require human approval before proceeding
          - when: node-starting
            then: require-approval

      - name: deploy
        type: execution
        step: deploy-model

    edges:
      - [train.output.model*, evaluate.input.model]
      - [evaluate.output.*, deploy.input.model]
```

#### Action Reference

**When (triggers):**
- `node-starting` - Before the node begins
- `node-complete` - After the node succeeds
- `node-error` - When the node fails

**If (conditions):**
- `metadata.metric_name >= value` - Based on logged metric
- `metadata.metric_name < value`
- Can reference any metadata key logged via JSON printing

**Then (actions):**
- `stop-pipeline` - Halt the entire pipeline
- `require-approval` - Pause until human approves

### Passing Metrics as Parameters

```yaml
edges:
  # Pass a logged metric from training as a parameter to fine-tuning
  - [train.metadata.best_learning_rate, finetune.parameter.learning_rate]

  # Pass a parameter value between nodes
  - [train.parameter.user_id, evaluate.parameter.user_id]
```

### Pipeline-Level Parameters

```yaml
- pipeline:
    name: configurable-pipeline
    parameters:
      - name: learning_rate
        default: 0.001
        targets:
          - train.parameter.learning_rate
          - finetune.parameter.learning_rate
      - name: dataset_version
        default: v2
        target: preprocess.parameter.version
    nodes:
      # ... nodes ...
    edges:
      # ... edges ...
```

### Deployment Node

```yaml
- pipeline:
    name: train-and-deploy
    nodes:
      - name: train
        type: execution
        step: train-model

      - name: deploy
        type: deployment
        deployment: my-deployment
        endpoints:
          - predict
        actions:
          - when: node-starting
            then: require-approval

    edges:
      - [train.output.model*, deploy.file.predict.model]
```

### Error Handling

```yaml
nodes:
  - name: train
    type: execution
    step: train-model
    on-error: stop-all    # stop-all | stop-next | continue | retry
```

| Strategy | Description |
|----------|-------------|
| `stop-all` | Stop the entire pipeline |
| `stop-next` | Don't start downstream nodes, but let parallel nodes finish |
| `continue` | Ignore the error and continue |
| `retry` | Retry the failed node |

### Node Reuse (Caching)

Skip nodes that already ran successfully with identical configuration:

```yaml
- pipeline:
    name: cached-pipeline
    reuse-executions: true
    nodes:
      # ... nodes ...
    edges:
      # ... edges ...
```

A node is considered unchanged when: same inputs, same command, same commit hash, same parameters, same step configuration.

## How to Identify Pipeline Opportunities

When analyzing a codebase, look for:

1. **Sequential scripts**: `run_preprocess.sh && run_train.sh && run_eval.sh` → pipeline
2. **Shared data between scripts**: Output of one script is input to another → edges
3. **Multiple model training runs**: Same training script with different configs → parallel nodes or task node
4. **Manual model selection**: Comparing results and picking the best → conditional actions + metadata edges
5. **Deployment after training**: Manual model promotion → deployment node with approval gate
6. **Scheduled retraining**: Cron jobs running training → pipeline triggers
7. **Data validation before training**: Data quality checks → conditional node with stop-pipeline action
8. **A/B model evaluation**: Testing multiple approaches → parallel training + comparison node

## Validate the Pipeline YAML

**IMPORTANT**: After defining a pipeline in `valohai.yaml`, always run `vh lint` to validate before running:

```shell
vh lint
```

This catches issues like:
- References to undefined steps in nodes
- Invalid edge syntax or missing node names
- Duplicate node names
- Missing required fields (`name`, `nodes`, `edges`)
- Invalid action conditions

Fix all lint errors before attempting to run the pipeline.

## Running Pipelines

```shell
# Run with local code
vh pipeline run training-pipeline --adhoc

# Run from Git
vh pipeline run training-pipeline

# Override node parameters
vh pipeline run training-pipeline --adhoc --train.epochs=200

# Override pipeline-level parameters
vh pipeline run training-pipeline --adhoc --learning_rate=0.0001
```

## Edge Cases

- Every node must reference an existing step defined in the same `valohai.yaml`
- Edges require both `nodes` and `edges` sections in the pipeline
- When using `override` on a node to clear an input, list the input name without a default
- Output wildcards (`*`) match against the filename, not the full path
- A pipeline must have at least one node and one edge
- Nodes without incoming edges start immediately when the pipeline runs
- Multiple edges can target the same input (files are merged into the input directory)
- `reuse-executions` only works when the pipeline is run from a Git commit (not `--adhoc`)
