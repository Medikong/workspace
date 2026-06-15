---
name: gitdeploy
description: "Use for Medikong GitHub tag-based deployment work: explaining or running deploy tags, choosing service/changed/all deploy targets, selecting patch/minor/major bumps, updating image-publish workflows, checking GitOps image tag updates, or maintaining deployment runbooks. If the deploy target is not explicit, ask the user to choose the target before proceeding."
---

# Git Deploy

## Purpose

Manage Medikong GitHub deployment work consistently through the tag-based image publish process.

## Required Reads

Before advising on or running a deployment, read:

- `workspace/docs/runbooks/deployment/tag-based-image-deploy.md`

When the user asks about design, rationale, or workflow changes, also read:

- `workspace/docs/architecture/deployment/README.md`

## Clarify Target

If the user has not clearly specified what to deploy, ask for the deploy target before proceeding.

Ask for:

- `SERVICE`: one service name, `changed`, or `all`
- `BUMP`: `patch`, `minor`, or `major` when creating a deploy tag

Do not assume `all`. Prefer `changed` for routine grouped deploys when the user asks to deploy changed services.

## Repo Boundaries

- `service`: Taskfile helper, deploy tag creation, image publish workflow
- `gitops`: image tag values and Argo CD deployment declarations
- `workspace`: shared design docs, runbooks, meeting notes, agent guidance

Before editing or running deployment commands, check the relevant repo status with `git status --short --branch`.

## Workflow

1. Identify whether the request is explanation, documentation, implementation, or an actual deploy.
2. Read the runbook first; read the architecture doc when design context is needed.
3. If deployment target or bump is unclear, ask a targeted question.
4. For actual deploys, follow the runbook commands instead of inventing a new sequence.
5. Preserve unrelated local changes and do not push deploy tags unless the user clearly asked to execute the deployment.

## Implementation Notes

- Prefer `SERVICE=changed` for routine grouped deploys.
- Keep `SERVICE=all` for forced full deploys only.
- Use `DRY_RUN=true` to preview the deploy tag and service plan before creating or pushing a tag.
- `changed` and `all` deploy tags must be annotated tags with the service deploy plan JSON in the tag message.
- The image publish workflow reads `{image, tag}` from the deploy tag or annotation; GitOps values updates must follow that per-service tag plan.
- The dashboard image is outside this tag-based service deploy path unless a future workflow explicitly adds it.
