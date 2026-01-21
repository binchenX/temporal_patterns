# Workflow Communication Patterns

Foundation patterns for interacting with workflows - Queries, Signals, Updates.

---

## Pattern 1: Query Pattern

### Problem
How do you read workflow state without modifying it? (Check order status, get progress, inspect current state)

### Solution
**Query**: Synchronous, read-only request to a workflow. Returns data immediately.

### Diagram
```
External System                    Workflow
     │                                │
     │   Query("getStatus")           │
     ├───────────────────────────────>│
     │                                │ Reads current state
     │                                │ (no state change)
     │   Returns: "processing"        │
     │<───────────────────────────────┤
     │                                │
```

### Code Example

**Workflow that handles queries:**
```go
type OrderState struct {
    Status       string
    Items        []string
    TotalAmount  float64
    LastUpdated  time.Time
}

func OrderWorkflow(ctx workflow.Context, orderID string) error {
    state := OrderState{
        Status: "pending",
        Items: []string{},
    }

    // Register query handler
    err := workflow.SetQueryHandler(ctx, "getStatus", func() (OrderState, error) {
        return state, nil
    })
    if err != nil {
        return err
    }

    // Workflow logic
    state.Status = "processing"
    workflow.ExecuteActivity(ctx, ProcessOrder, orderID).Get(ctx, nil)

    state.Status = "completed"
    state.LastUpdated = workflow.Now(ctx)

    workflow.Sleep(ctx, 24*time.Hour) // Long-running
    return nil
}
```

**Client querying workflow:**
```go
client, _ := client.Dial(client.Options{...})

// Query the workflow
response, err := client.QueryWorkflow(ctx, "order-123", "", "getStatus")
if err != nil {
    log.Fatal(err)
}

var state OrderState
response.Get(&state)
fmt.Printf("Order status: %s\n", state.Status)
```

### When to Use
- Displaying workflow status in UI
- Monitoring/dashboards
- Debugging (inspect internal state)
- Read-only operations

### When NOT to Use
- Need to modify workflow state → Use **Signal** or **Update** instead
- Need to trigger workflow logic or activities → Queries are read-only
- Need guaranteed consistency with external state → Queries don't block workflow progress

### Characteristics
- ✅ Synchronous - immediate response
- ✅ Read-only - cannot modify workflow state
- ✅ No workflow history entry
- ✅ Works on completed workflows too
- ❌ Cannot call activities
- ❌ Cannot wait or use timers


## Pattern 2: Signal Pattern

### Problem
How do you send data to a running workflow from outside? (User clicks button, webhook arrives, external event occurs)

### Solution
**Signal**: Asynchronous, fire-and-forget message to a workflow. Doesn't wait for response.

### Diagram
```
External System                    Workflow
     │                                │
     │   Signal("approve")            │
     ├───────────────────────────────>│
     │                                │ (continues processing)
     │   Returns immediately          │ (handles signal when ready)
     │<───────────────────────────────│
     │                                │

Time passes...
     │                                │
     │                                │ Signal handler executes
     │                                │ Workflow state changes
```

### Code Example

**Workflow that receives signals:**
```go
func OrderWorkflow(ctx workflow.Context, orderID string) error {
    var approved bool
    var cancelled bool

    // Register signal handlers
    workflow.SetSignalHandler(ctx, "approve", func() {
        approved = true
    })

    workflow.SetSignalHandler(ctx, "cancel", func() {
        cancelled = true
    })

    // Wait for approval or cancellation
    workflow.Await(ctx, func() bool {
        return approved || cancelled
    })

    if cancelled {
        return errors.New("order cancelled")
    }

    // Continue with approved order
    var result string
    workflow.ExecuteActivity(ctx, ProcessOrder, orderID).Get(ctx, &result)
    return nil
}
```

**Client sending signal:**
```go
// Start workflow
client, _ := client.Dial(client.Options{...})
we, _ := client.ExecuteWorkflow(ctx,
    client.StartWorkflowOptions{
        ID: "order-123",
        TaskQueue: "orders",
    },
    OrderWorkflow,
    "order-123",
)

// Later... user clicks "Approve" button
err := client.SignalWorkflow(ctx, "order-123", "", "approve", nil)

// Or user clicks "Cancel"
err := client.SignalWorkflow(ctx, "order-123", "", "cancel", nil)
```

### When to Use
- User actions (approve, cancel, update preferences)
- Webhooks from external systems
- Events that don't need immediate response
- Multiple signals over workflow lifetime

### When NOT to Use
- Need immediate confirmation that change was applied → Use **Update** instead
- Need to validate input before accepting → Use **Update** instead
- Just need to read workflow state → Use **Query** instead
- Workflow already completed → Signal will fail

### Characteristics
- ✅ Asynchronous - sender doesn't wait
- ✅ Can send multiple signals to same workflow
- ✅ Signals are queued if workflow is busy
- ❌ No return value
- ❌ No validation at send time

---

---

## Pattern 3: Update Pattern

### Problem
How do you modify workflow state synchronously with validation? (Change shipping address, update quantity, modify settings with immediate confirmation)

### Solution
**Update**: Synchronous, validated state modification. Like Signal + Query combined.

### Diagram
```
External System                    Workflow
     │                                │
     │   Update("changeAddress", addr)│
     ├───────────────────────────────>│
     │                                │ Validates input
     │                                │ Updates state
     │                                │ Returns result
     │   Returns: "Success"           │
     │<───────────────────────────────┤
     │                                │
```

### Code Example

**Workflow that handles updates:**
```go
func OrderWorkflow(ctx workflow.Context, orderID string) error {
    var shippingAddress string = "123 Main St"
    var orderStatus string = "pending"

    // Register update handler
    err := workflow.SetUpdateHandler(ctx, "changeAddress", func(newAddress string) (string, error) {
        // Validation
        if orderStatus == "shipped" {
            return "", errors.New("cannot change address: order already shipped")
        }
        if newAddress == "" {
            return "", errors.New("address cannot be empty")
        }

        // Update state
        shippingAddress = newAddress

        // Return result
        return fmt.Sprintf("Address updated to: %s", shippingAddress), nil
    })
    if err != nil {
        return err
    }

    // Process order...
    workflow.ExecuteActivity(ctx, ProcessOrder, orderID).Get(ctx, nil)
    orderStatus = "shipped"

    return nil
}
```

**Client sending update:**
```go
client, _ := client.Dial(client.Options{...})

// Send update and wait for result
updateHandle, err := client.UpdateWorkflow(ctx,
    client.UpdateWorkflowOptions{
        WorkflowID: "order-123",
        UpdateName: "changeAddress",
        Args:       []interface{}{"456 Oak Ave"},
    },
)
if err != nil {
    log.Fatal(err)
}

// Get result
var result string
err = updateHandle.Get(ctx, &result)
if err != nil {
    // Validation failed or update rejected
    log.Printf("Update failed: %v", err)
} else {
    log.Printf("Update succeeded: %s", result)
}
```

### When to Use
- State changes that need validation
- Operations that must confirm success/failure immediately
- When you need stronger guarantees than Signal
- Replacing Signal + Query pattern

### When NOT to Use
- Fire-and-forget operations → Use **Signal** instead (simpler)
- Just reading state → Use **Query** instead
- Workflow already completed → Update will fail
- Don't need synchronous response → Signal is more efficient

### Characteristics
- ✅ Synchronous - waits for completion
- ✅ Can validate input
- ✅ Returns success/failure immediately
- ✅ Recorded in workflow history
- ✅ Can modify workflow state
- ❌ More complex than Signal
- ❌ Workflow must be running (fails if workflow completed)

---


## Comparison Table

| Feature | Query | Signal | Update |
|---------|-------|--------|--------|
| **Synchronous?** | ✅ Yes | ❌ No | ✅ Yes |
| **Returns value?** | ✅ Yes | ❌ No | ✅ Yes |
| **Modifies state?** | ❌ No | ✅ Yes | ✅ Yes |
| **Can validate?** | N/A | ❌ No | ✅ Yes |
| **In history?** | ❌ No | ✅ Yes | ✅ Yes |
| **Works on completed?** | ✅ Yes | ❌ No | ❌ No |
| **Complexity** | Low | Low | Medium | 

---

## Common Patterns Combined

### Pattern: Signal + Await
```go
// Wait for approval signal
var approved bool
workflow.SetSignalHandler(ctx, "approve", func() { approved = true })
workflow.Await(ctx, func() bool { return approved })
```

### Pattern: Update with Validation
```go
workflow.SetUpdateHandler(ctx, "updateQuantity", func(newQty int) error {
    if newQty <= 0 {
        return errors.New("quantity must be positive")
    }
    if orderStatus == "shipped" {
        return errors.New("cannot modify shipped order")
    }
    quantity = newQty
    return nil
})
```

### Pattern: Query Current Progress
```go
workflow.SetQueryHandler(ctx, "getProgress", func() float64 {
    return float64(itemsProcessed) / float64(totalItems) * 100.0
})
```

---

## Quick Decision Guide

**Need to send data to workflow?**
- Doesn't need response → **Signal**
- Needs response + validation → **Update**

**Need to read workflow state?**
- Just read, no changes → **Query**

**Need to break down workflow?**
- Same namespace, reusable logic → **Child Workflow**
- Different namespace → **Nexus** (see next pattern doc)

**Need both read and write?**
- Modern way → **Update**
- Legacy way → Signal + Query (avoid this)

---

## Real-World Example: Order Workflow

```go
func OrderWorkflow(ctx workflow.Context, order Order) error {
    state := "pending"

    // Query: Check status
    workflow.SetQueryHandler(ctx, "status", func() string {
        return state
    })

    // Signal: Cancel order
    var cancelRequested bool
    workflow.SetSignalHandler(ctx, "cancel", func() {
        cancelRequested = true
    })

    // Update: Change shipping address
    workflow.SetUpdateHandler(ctx, "updateAddress", func(addr string) error {
        if state == "shipped" {
            return errors.New("too late to change")
        }
        order.ShippingAddress = addr
        return nil
    })

    // Process payment (child workflow)
    state = "processing_payment"
    var paymentID string
    childCtx := workflow.WithChildOptions(ctx, workflow.ChildWorkflowOptions{
        WorkflowID: fmt.Sprintf("payment-%s", order.ID),
    })
    err := workflow.ExecuteChildWorkflow(childCtx, PaymentWorkflow, order.Amount).Get(ctx, &paymentID)
    if err != nil {
        state = "payment_failed"
        return err
    }

    // Check for cancellation
    if cancelRequested {
        state = "cancelled"
        workflow.ExecuteActivity(ctx, RefundPayment, paymentID).Get(ctx, nil)
        return nil
    }

    // Ship order
    state = "shipping"
    workflow.ExecuteActivity(ctx, ShipOrder, order).Get(ctx, nil)

    state = "completed"
    return nil
}
```

---

