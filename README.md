# Valohai Agent Skills

A collection of [Agent Skills](https://agentskills.io/specification) for migrating ML codebases to [Valohai](https://valohai.com), the MLOps platform.

These skills give AI coding agents (Claude Code, GitHub Copilot, Cursor, etc.) the knowledge to help users migrate their existing ML projects to Valohai with minimal code changes.

## Skills

| Skill | Description |
|-------|-------------|
| [`valohai-migrate-parameters`](./valohai-migrate-parameters/) | Convert hardcoded values to Valohai-managed parameters with argparse + YAML |
| [`valohai-migrate-metrics`](./valohai-migrate-metrics/) | Add JSON-based metrics tracking for experiment comparison and visualization |
| [`valohai-migrate-data`](./valohai-migrate-data/) | Migrate data loading/saving to Valohai's input/output system |
| [`valohai-yaml-step`](./valohai-yaml-step/) | Create `valohai.yaml` step definitions with Docker images, commands, and configuration |
| [`valohai-project-run`](./valohai-project-run/) | Set up Valohai projects and run executions/pipelines via the CLI |
| [`valohai-design-pipelines`](./valohai-design-pipelines/) | Analyze ML projects and design multi-step Valohai pipelines |

## Installation

```shell
npx skills add valohai/valohai-skills
```

## Migration Workflow

The recommended order for migrating an existing ML project:

1. **Analyze** the project structure and identify scripts, dependencies, and data flow
2. **Create steps** (`valohai-yaml-step`) - Define Docker image, commands, and basic YAML
3. **Add parameters** (`valohai-migrate-parameters`) - Make hyperparameters configurable
4. **Add metrics** (`valohai-migrate-metrics`) - Enable experiment tracking with JSON printing
5. **Migrate data** (`valohai-migrate-data`) - Set up inputs from cloud storage and outputs to `/valohai/outputs/`
6. **Set up project** (`valohai-project-run`) - Create project, link directory, run first execution
7. **Design pipelines** (`valohai-design-pipelines`) - Connect steps into automated workflows

## Philosophy

Valohai follows a **YAML-over-SDK** approach: your ML code should not be entangled with the platform. Configuration lives in `valohai.yaml`, and your code remains portable and framework-agnostic.

## License

Apache-2.0
