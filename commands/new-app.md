---
description: Create a new Kinotic application end to end
---

Create a new Kinotic application by following the workflow in the kinotic `create-app`
skill exactly: help the user sign up for Kinotic OS (or sign in) and connect its MCP
server (`kinotic-os`, or `kinotic-local` for a local Kinotic), create the Application and
its first Project, handle repository provisioning (and GitHub linking on Kinotic Cloud),
clone the repository with a Kinotic token, verify the scaffold, define a first entity, and
push — then confirm with `Project Service Find Deployment` that the deployment's
`state.observed` reached `RUNNING` at the pushed commit.

Use `$ARGUMENTS` as the application name if provided; otherwise ask the user for a name
and a one-line description before starting.
