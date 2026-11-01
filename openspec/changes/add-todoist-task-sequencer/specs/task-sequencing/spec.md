# Spec Delta

## Purpose

Keeps exactly one task from a configured Todoist project on the user's "Today" view. When the current task is completed or deleted, the next task in the user's manual order gets today's date.

## ADDED Requirements

### Requirement: Sequencing is scoped to one configured project
The system SHALL act on a single Todoist project, identified by a project ID supplied in deployment configuration. It MUST NOT read or change tasks in any other project.

#### Scenario: Event from the configured project
- **WHEN** a qualifying event arrives for a task whose project is the configured project
- **THEN** the system evaluates the sequencing rule for that project

#### Scenario: Event from another project
- **WHEN** a qualifying event arrives for a task in any other project
- **THEN** the system makes no changes to any task

### Requirement: Completion and deletion trigger the sequencing rule
The system SHALL evaluate the sequencing rule when a task in the configured project is completed or deleted. Other task changes (creation, edits, rescheduling, adding or removing a date, uncompleting) MUST NOT trigger the rule.

#### Scenario: Task completed
- **WHEN** the user completes a task in the configured project
- **THEN** the system evaluates the sequencing rule

#### Scenario: Task deleted
- **WHEN** the user deletes a task in the configured project
- **THEN** the system evaluates the sequencing rule

#### Scenario: Manual date removal is ignored
- **WHEN** the user removes the due date from the currently dated task without completing or deleting it
- **THEN** the system makes no changes, even though no task in the project has a date

#### Scenario: Manual reschedule is ignored
- **WHEN** the user changes the due date of any task in the configured project
- **THEN** the system makes no changes

### Requirement: Next task receives today's date only when no task is dated
When the rule runs, the system SHALL check every open task in the configured project (including subtasks) for a due date. If any open task has one, the system MUST make no changes. If none does, the system SHALL give the next task a due date of today. "Today" is the current date in the user's Todoist timezone, and the due date has no time.

#### Scenario: Current task completed, next task promoted
- **WHEN** the dated task is completed and the remaining open tasks have no due dates
- **THEN** the next task in manual order gets a due date of today and appears in the user's "Today" view

#### Scenario: Another task still has a date
- **WHEN** a task is completed but another open task in the project still has a due date (for example, one the user scheduled by hand)
- **THEN** the system makes no changes

#### Scenario: Undated task completed
- **WHEN** the user completes a task that had no due date while another task in the project is still dated
- **THEN** the system makes no changes

#### Scenario: Project is empty
- **WHEN** the rule runs and the project has no open tasks
- **THEN** the system makes no changes and finishes without error

#### Scenario: Recurring task completed
- **WHEN** the user completes an occurrence of a recurring task, so Todoist moves its due date to the next occurrence and the task stays open
- **THEN** the system sees that the task still has a due date and makes no changes

### Requirement: Next task follows the user's manual order
The system SHALL pick the next task from open top-level tasks (tasks with no parent) in the project. It SHALL use the order Todoist shows in the project's list view: first the tasks not in any section, in their manual order, then each section in section order with that section's tasks in manual order. Subtasks MUST NOT be picked.

#### Scenario: Unsectioned tasks come first
- **WHEN** the project has unsectioned tasks and tasks in sections, and the rule promotes a task
- **THEN** the topmost unsectioned task is promoted

#### Scenario: Sections in order
- **WHEN** the project has no unsectioned open tasks and the rule promotes a task
- **THEN** the topmost task of the first section that has open tasks is promoted

#### Scenario: Subtasks are skipped
- **WHEN** the topmost open item in the list is a subtask whose parent is already completed or somewhere else in the list
- **THEN** the system promotes the topmost top-level task instead

### Requirement: Sequencing is idempotent
The system SHALL leave the same end state when it processes the same event more than once, or several events close together. At most one task MUST end up with a date assigned by the system.

#### Scenario: Duplicate webhook delivery
- **WHEN** Todoist delivers the same completion event twice
- **THEN** the first delivery promotes the next task and the second makes no changes, because a dated task now exists

#### Scenario: Two completions in quick succession
- **WHEN** two tasks in the project are completed seconds apart
- **THEN** exactly one open task ends up dated today after both events are processed

### Requirement: Failures are surfaced for retry
If the system can't read or update tasks (for example, the Todoist API is unavailable, rate-limited, or rejects the credentials), it SHALL report the processing as failed so the delivery can be retried, and it SHALL log the reason. It MUST NOT leave partially applied changes beyond a single task's due date.

#### Scenario: Todoist API unavailable
- **WHEN** the Todoist API returns a server error while the rule runs
- **THEN** the system reports failure for that delivery and logs the error

#### Scenario: Invalid access token
- **WHEN** the Todoist API rejects the stored access token
- **THEN** the system reports failure and logs an authorization error without including the token
