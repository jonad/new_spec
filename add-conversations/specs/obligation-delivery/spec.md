# Spec Delta

## Purpose

Delivers the named obligations a command committed with its receipt, after commit, through claims that survive crashes and a reconciler that re-drives what was not delivered, so that no delivery is fire-and-forget and lag stays visible.

## ADDED Requirements

### Requirement: Every obligation has a delivery record

Each named obligation SHALL have exactly one delivery record, created in the obligation's transaction with status `queued`, holding the owner, generation and expiry of its current claim, the attempt count, the delivery time and the last error.
The obligation itself SHALL stay immutable.

#### Scenario: Record created with the obligation

- **WHEN** a command that names an obligation commits
- **THEN** a delivery record for that obligation exists with status `queued`, no owner and zero attempts

#### Scenario: Failure before commit

- **WHEN** a command fails before commit
- **THEN** no obligation and no delivery record from it exist

### Requirement: Delivery happens after commit through a claim

After a command commits, the deliverer SHALL claim each of its queued delivery records with an owner, a new generation and an expiry, hand the obligation to the consumer registered for its kind, and mark the record `delivered` with the time.
Nothing SHALL be delivered inside the command's transaction, and a claim SHALL be honoured only while it is the current, unexpired one.

#### Scenario: Delivered right after commit

- **WHEN** a command naming a hint obligation commits and the deliverer runs
- **THEN** the consumer receives the hint once and the delivery record is `delivered` with one attempt

#### Scenario: Expired claim is not honoured

- **WHEN** a deliverer claims a record, its claim expires before it finishes, and the reconciler re-claims the record
- **THEN** the first deliverer's late mark is refused and only the holder of the current claim can mark the record delivered

### Requirement: The reconciler re-drives undelivered obligations

A reconciler SHALL periodically find records that are `queued`, or `claimed` with an expired claim, and deliver them with bounded backoff, incrementing the attempt count and keeping the last error.

#### Scenario: Deliverer crashes after commit

- **WHEN** a command commits and the process dies before the deliverer runs
- **THEN** the reconciler delivers the obligation on its next pass and the consumer receives it exactly once

#### Scenario: Consumer fails

- **WHEN** a consumer raises on delivery
- **THEN** the record returns to `queued` with the attempt count incremented and the error kept, and a later pass delivers it

### Requirement: Kinds without a consumer stay queued and visible

An obligation whose kind has no registered consumer SHALL stay `queued`, SHALL NOT count as failed, and SHALL be reported by the lag view with its kind and age.

#### Scenario: Ledger obligation before its consumer exists

- **WHEN** `initializeWorkspace` commits its `workspace.initialized` obligation and no ledger consumer is registered
- **THEN** the record stays `queued` with zero attempts and the lag view lists it

### Requirement: Delivery never changes a command's response

A command's response SHALL depend only on its committed outcome.
A delivery failure, a slow consumer or an absent consumer SHALL NOT change the status or body returned to the caller.

#### Scenario: Consumer failure after a successful command

- **WHEN** a `postMessage` commits and its hint consumer fails
- **THEN** the caller still receives `201` with the message, and the hint is re-driven later
