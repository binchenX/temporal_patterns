# Child Workflow Patterns

A **Child Workflow** is a Workflow Execution spawned from within another Workflow (the "parent") in the **same namespace**. Think of it as a structured sub-process with its own lifecycle, state, and event history.

Before you go into details about the use cases and patterns of child workflows, we need to get one thing out of the way - the distinction between Activity and Child workflow. Some of these things may not be clear initially but they will become clearer by the end of the doc.

## Child Workflow vs Activity

### Key Differences

| Feature | Activity | Child Workflow |
|---------|----------|----------------|
| **Can call activities?** | ❌ No | ✅ Yes |
| **Can receive signals?** | ❌ No | ✅ Yes |
| **Can be signal/queried?** | ❌ No | ✅ Yes |
| **Deterministic?** | ❌ No | ✅ Yes |
| **Can use timers?** | ❌ No | ✅ Yes |
| **Parent cancels → Child?** | Always canceled | Depends on policy |
| **Event history** | Single activity event | Full workflow history, indepent from parent |

---

### Pattern 1: Service Boundary Isolation

**When to use:** Different teams/services need to collaborate, each maintaining their own workflow logic and workers.

**Problem:** Order workflow needs payment logic, but payment is owned by a different team running separate workers.

**Solution:** Use child workflow on different task queue to create clear service boundary.

```go
// Order team's workflow
func OrderWorkflow(ctx workflow.Context, order Order) error {
    // Payment handled by payment team's workers
    childCtx := workflow.WithChildOptions(ctx, workflow.ChildWorkflowOptions{
        TaskQueue: "PAYMENT_QUEUE",  // Different task queue
        WorkflowID: fmt.Sprintf("payment-%s", order.ID),
    })

    var paymentID string
    err := workflow.ExecuteChildWorkflow(childCtx, PaymentWorkflow, order.Amount).Get(ctx, &paymentID)
    return err
}
```

**Why it works:**
- Payment team controls PaymentWorkflow code
- Order team doesn't need payment worker code
- Clear service boundary

---

### Pattern 2: Large Workload Partitioning

**When to use:** Need to process more activities than a single workflow's event history can handle.

**Problem:** Single workflow has Event History limit (50K events). Need to process 100K+ activities.

#### The Problem - Why Single Workflow Fails

```go
// ❌ THIS FAILS - Single workflow trying to process 100K orders
func ProcessAllOrdersWorkflow(ctx workflow.Context, orderIDs []string) error {
    // orderIDs has 100,000 entries

    for _, orderID := range orderIDs {
        // Each activity creates ~5 events in history:
        // - ActivityTaskScheduled
        // - ActivityTaskStarted
        // - ActivityTaskCompleted
        // - (plus retry events if it fails)
        var result string
        workflow.ExecuteActivity(ctx, ProcessOrder, orderID).Get(ctx, &result)
    }

    // Problem: 100K activities × 5 events = 500K events
    // Limit: 50K events per workflow
    // Result: Workflow TERMINATED by Temporal!
    return nil
}
```

#### Solution - Partition with Child Workflows

```go
// ✅ Parent workflow - stays under event limit
func ProcessAllOrdersWorkflow(ctx workflow.Context, orderIDs []string) error {
    // Split 100K orders into batches of 1000
    batchSize := 1000
    numBatches := len(orderIDs) / batchSize  // 100 batches

    var futures []workflow.ChildWorkflowFuture

    for i := 0; i < numBatches; i++ {
        start := i * batchSize
        end := start + batchSize
        batch := orderIDs[start:end]

        childCtx := workflow.WithChildOptions(ctx, workflow.ChildWorkflowOptions{
            WorkflowID: fmt.Sprintf("batch-%d", i),
        })

        // Start child workflow for this batch
        future := workflow.ExecuteChildWorkflow(childCtx, ProcessBatchWorkflow, batch)
        futures = append(futures, future)
    }

    // Wait for all batches to complete (all children run in parallel)
    for i, future := range futures {
        err := future.Get(ctx, nil)
        if err != nil {
            return fmt.Errorf("batch %d failed: %w", i, err)
        }
    }

    return nil
}

// Child workflow - processes one batch
func ProcessBatchWorkflow(ctx workflow.Context, orderIDs []string) error {
    // This workflow processes 1000 orders
    // Creates ~5000 events in its OWN history (under 50K limit)

    for _, orderID := range orderIDs {
        var result string
        err := workflow.ExecuteActivity(ctx, ProcessOrder, orderID).Get(ctx, &result)
        if err != nil {
            return err
        }
    }

    return nil
}
```

## Pattern 3: "Private" Workflows

**When to use:** Ensure that the workflow can only be triggered by a parent workflow within a namespace, but not from a client outside.

**Problem:** In class design we have protected/private methods, which can be used only by methods in the same class. We may have the same need for workflow design. We want a workflow that can only be triggered by a parent workflow within the same namespace.

**Solution:** Check Parent Workflow Execution

```go
func PaymentWorkflow(ctx workflow.Context, req PaymentRequest) (PaymentResult, error) {
    info := workflow.GetInfo(ctx)

    // Enforce: Must be started as child workflow
    if info.ParentWorkflowExecution == nil {
        return PaymentResult{}, errors.New("PaymentWorkflow must be started as child workflow")
    }

    // Optionally: Enforce specific parent type
    // if info.ParentWorkflowExecution.WorkflowType != "OrderWorkflow" {
    //     return PaymentResult{}, errors.New("PaymentWorkflow must be started from OrderWorkflow")
    // }

    // Continue with workflow logic...
}
```
---

## Additional Resources

- [Temporal Child Workflow Docs](https://docs.temporal.io/child-workflows)
- [Parent Close Policy](https://docs.temporal.io/parent-close-policy)
- [Go SDK Child Workflow Guide](https://docs.temporal.io/develop/go/child-workflows)
