---
description: Create a new Kinotic application end to end
---

Create a new Kinotic application by following the workflow in the kinotic `create-app`
skill exactly: help the user sign up for Kinotic OS (or sign in) and connect its MCP
server (`kinotic-os`, or `kinotic-local` for a local Kinotic), create the Application and
its first Project, handle repository provisioning (and GitHub linking on Kinotic Cloud),
clone the repository with a Kinotic token, verify the scaffold, define a first entity, and
push — then confirm with `Project Service Find Deployment` that the deployment's
`state.observed` reached `RUNNING` at the pushed commit. The scaffold has no microservice;
when the project has one, check it with `Microservice Deployment Service Find All For
Project` and report any failureMessage. If a step fails with a message about the Kinotic beta
cloud, explain the beta limits as the skill's **Beta limits** section says.
A running VM is not proof the service answers: verify an application operation through
its public interface when one exists. Follow the deploying skill's superseded-push handling.

Use `$ARGUMENTS` as the application name if provided; otherwise ask the user for a name
and a one-line description before starting.
