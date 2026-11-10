# 0043. Desired commits

Status: accepted

## Context

Chapter 13 has the status of each instance name `desired`, the commit of the desired value its reconciler acts on, which a plan step waits on. `supervisor` reads each specification from the store, whose nodes name the commit that last wrote them, and restarts an instance when a commit changes its configuration record; the runner publishes each instance's record.

## Decision

- After bringing an instance to its specification, `supervisor` gives the runner that specification's version through `Control.converge`, and after a restart for a configuration record, the commit that changed it.
- The runner keeps the greatest commit it was given for the instance and publishes it in the instance's status.

## Consequences

- An instance whose reconciliation is under way shows the commit of the previous pass until it ends.
