# Business Process Flow Patterns

This document explains three key patterns for orchestrating business processes with Temporal: Approval, Delayed Start, and Scheduled Workflows.

## 1. Approval Pattern - Human-in-the-loop workflows

### Overview

**What it does:** Pauses workflow execution until a human makes a decision (approve/reject/escalate).

### How It Works

```java
public class ApprovalWorkflowImpl implements ApprovalWorkflow {
  private ApprovalData approvalData;

  @Override
  public String execute(String requestId, Duration timeout) {
    // Workflow blocks here waiting for signal or timeout
    boolean approved = Workflow.await(timeout, () -> approvalData != null);

    if (approved) {
      return "Approved by " + approvalData.getApprover();
    } else {
      return "Approval timeout - auto-rejected";
    }
  }

  @Override
  public void submitApproval(ApprovalData data) {
    this.approvalData = data; // Signal handler receives approval
  }
}
```

### Key Features

- Uses `Workflow.await()` to block until signal received
- Captures rich approval data (approver, decision, comments, timestamp)
- Supports timeouts (auto-reject after X hours/days)
- Can implement multi-level approvals (L1 → L2 → L3)
- Can escalate to managers if initial approval times out

### Use Cases

- Purchase order approvals
- Expense report reviews
- Code deployment gates
- Contract signing workflows
- Access request approvals

---

## 2. Delayed Start Pattern

### Overview

**What it does:** Creates a workflow immediately but waits before executing the main logic.

### How It Works

```java
public class DelayedStartWorkflowImpl implements DelayedStartWorkflow {
  @Override
  public String execute(String taskId, Duration delay) {
    // Workflow exists but sleeps first
    Workflow.sleep(delay);

    // Now execute the actual work
    return activities.processTask(taskId);
  }
}
```

### Key Features

- Workflow is created immediately (gets workflow ID, appears in UI)
- `Workflow.sleep()` defers execution until delay expires
- Can be cancelled during delay period
- Can query workflow status while waiting
- Different from scheduled workflows (one-time vs recurring)

### Use Cases

- Grace periods (delete account in 30 days unless cancelled)
- Reminder notifications (send email in 1 hour)
- Delayed order processing (ship tomorrow morning)
- Trial period expirations (downgrade after 14 days)
- Scheduled one-time events (maintenance window in 2 hours)

### Example

```java
// Create workflow to delete account in 30 days
WorkflowOptions options = WorkflowOptions.newBuilder()
    .setWorkflowId("delete-account-" + accountId)
    .build();

DeleteAccountWorkflow workflow = client.newWorkflowStub(
    DeleteAccountWorkflow.class, options);

// Starts immediately but waits 30 days
WorkflowClient.start(workflow::execute, accountId, Duration.ofDays(30));

// User can cancel anytime during 30 days
workflow.cancel(); // Cancels the deletion
```

---

## 3. Scheduled Workflows Pattern

### Overview

**What it does:** Executes workflows automatically on recurring schedules (like cron jobs, but durable).

### How It Works

```java
// Create a schedule (not a workflow)
Schedule schedule = Schedule.newBuilder()
    .setAction(
        ScheduleActionStartWorkflow.newBuilder()
            .setWorkflowType(BackupWorkflow.class)
            .setTaskQueue("backups")
            .build())
    .setSpec(
        ScheduleSpec.newBuilder()
            .setCronExpression("0 2 * * *") // 2 AM daily
            .build())
    .build();

client.createSchedule("daily-backup", schedule);
```

### Key Features

- Runs workflows on cron-like schedules
- Supports intervals (`@every 1h`), cron expressions, or calendar specs
- Handles missed runs (backfill or skip)
- Can pause/unpause schedules
- Each execution gets unique workflow ID
- Survives server restarts (durable)

### Schedule Types

```java
// Cron expression (Unix cron syntax)
.setCronExpression("0 */6 * * *") // Every 6 hours

// Interval
.setInterval(Duration.ofHours(1)) // Every hour

// Calendar-based
.setCalendar(
    CalendarSpec.newBuilder()
        .setDayOfWeek(Range.newBuilder().setStart(1).setEnd(5)) // Mon-Fri
        .setHour(9) // 9 AM
        .build())
```

### Use Cases

- Daily database backups
- Hourly report generation
- Monthly billing runs
- Weekly cleanup jobs
- Periodic health checks
- Nightly data synchronization

---

## Pattern Comparison

| Pattern | Trigger | Frequency | Cancellable | Use Case |
|---------|---------|-----------|-------------|----------|
| **Approval** | Human signal | One-time | Yes | Purchase order approval |
| **Delayed Start** | Timer (one-time) | One-time | Yes | Delete account in 30 days |
| **Scheduled Workflows** | Cron/interval | Recurring | Schedule can be paused | Daily backups |

### Key Differences

- **Approval** = waits for human decision
- **Delayed Start** = single workflow that waits, then executes once
- **Scheduled Workflows** = creates new workflow executions repeatedly on schedule


