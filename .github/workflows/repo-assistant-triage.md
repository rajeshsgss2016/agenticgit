---
on:
  workflow_dispatch:
    inputs:
      task:
        description: Task to implement in this repository
        required: true
        type: string
engine:
  id: copilot
  model: gpt-4.1

permissions:
  contents: read
  issues: read
  pull-requests: read
  copilot-requests: write

concurrency:
  job-discriminator: ${{ github.run_id }}

safe-outputs:
  create-pull-request:
    max: 1
---

# Repo Assistant

Implement the requested task in the checked-out repository and open one pull request with the changes.

Requested task: ${{ github.event.inputs.task }}

Make only the changes needed for the requested task. Run relevant tests or validation before opening the pull request. If the task cannot be completed safely or no code change is needed, explain why and do not open a pull request.