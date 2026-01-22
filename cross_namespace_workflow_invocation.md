# Cross-Namespace Workflow Invocation

**Namespace** in Temporal is a unit of isolation. It provides:

- **Team isolation**: Payment team has `payment-prod` namespace, Order team has `order-prod` namespace
- **Security boundaries**: Different access controls, certificates, and permissions per namespace
- **Failure isolation**: Issues in one namespace don't affect others.

But workflows in different namespaces cannot directly call each other like child workflows can (child workflows must be in the same namespace).

## Problem: Workflow Invocation Across Temporal Namespace

```
Namespace A (order-prod)          Namespace B (payment-prod)
┌─────────────────────┐          ┌──────────────────────┐
│ OrderWorkflow       │   ???    │ PaymentWorkflow      │
│ wants to invoke ────┼─────────>│                      │
│                     │          │                      │
└─────────────────────┘          └──────────────────────┘
```

---

## Pattern 1: Activity Wrapper

### Overview

Expose the target workflow through an HTTP/gRPC API, then call that API from an Activity.

```
Namespace A                      API Gateway              Namespace B
┌─────────────────────┐         ┌─────────────┐         ┌──────────────────────┐
│ OrderWorkflow       │         │             │         │                      │
│   │                 │         │             │         │ Worker polls         │
│   └─> Activity ─────┼────────>│  HTTP/gRPC  ├────────>│ PaymentWorkflow      │
│       (HTTP call)   │         │             │         │                      │
└─────────────────────┘         └─────────────┘         └──────────────────────┘
```

### How It Works

**Namespace B (Handler Side):**
1. Create HTTP/gRPC API endpoint that starts PaymentWorkflow
2. API endpoint uses Temporal client to start workflow in `payment-prod` namespace

**Namespace A (Caller Side):**
1. OrderWorkflow calls an Activity
2. Activity makes HTTP/gRPC call to Namespace B's API
3. Activity waits for response (or polls for completion)

---

## Code Example: Activity Wrapper Pattern

### Namespace B: Expose Workflow via API

```go
// API server in payment team's infrastructure
// Exposes PaymentWorkflow through HTTP endpoint

package main

import (
    "encoding/json"
    "net/http"
    "go.temporal.io/sdk/client"
)

type PaymentRequest struct {
    OrderID string  `json:"order_id"`
    Amount  float64 `json:"amount"`
}

type PaymentResponse struct {
    PaymentID string `json:"payment_id"`
    Status    string `json:"status"`
}

// HTTP handler that starts PaymentWorkflow
func handlePaymentRequest(w http.ResponseWriter, r *http.Request) {
    var req PaymentRequest
    json.NewDecoder(r.Body).Decode(&req)

    // Create Temporal client for payment-prod namespace
    c, err := client.Dial(client.Options{
        HostPort:  "payment-prod.abc123.tmprl.cloud:7233",
        Namespace: "payment-prod",
        // ... mTLS configuration
    })
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    defer c.Close()

    // Start PaymentWorkflow
    workflowOptions := client.StartWorkflowOptions{
        ID:        "payment-" + req.OrderID,
        TaskQueue: "payment-queue",
    }

    we, err := c.ExecuteWorkflow(context.Background(), workflowOptions, PaymentWorkflow, req)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    // Wait for workflow to complete (or return workflowID for async polling)
    var result PaymentResult
    err = we.Get(context.Background(), &result)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    response := PaymentResponse{
        PaymentID: result.PaymentID,
        Status:    "completed",
    }

    json.NewEncoder(w).Encode(response)
}

func main() {
    http.HandleFunc("/payment/process", handlePaymentRequest)
    http.ListenAndServe(":8080", nil)
}
```

### Namespace A: Call API from Activity

```go
// Activity in order team's workflow worker

package main

import (
    "bytes"
    "context"
    "encoding/json"
    "net/http"
    "time"
    "go.temporal.io/sdk/workflow"
)

type PaymentRequest struct {
    OrderID string  `json:"order_id"`
    Amount  float64 `json:"amount"`
}

type PaymentResponse struct {
    PaymentID string `json:"payment_id"`
    Status    string `json:"status"`
}

// Activity that calls payment API
func CallPaymentServiceActivity(ctx context.Context, req PaymentRequest) (PaymentResponse, error) {
    // Marshal request
    jsonData, err := json.Marshal(req)
    if err != nil {
        return PaymentResponse{}, err
    }

    // Call payment API
    httpReq, err := http.NewRequestWithContext(ctx, "POST",
        "https://payment-api.example.com/payment/process",
        bytes.NewBuffer(jsonData))
    if err != nil {
        return PaymentResponse{}, err
    }

    httpReq.Header.Set("Content-Type", "application/json")
    // Add authentication headers (API key, mTLS, etc.)

    client := &http.Client{Timeout: 30 * time.Second}
    resp, err := client.Do(httpReq)
    if err != nil {
        return PaymentResponse{}, err
    }
    defer resp.Body.Close()

    // Parse response
    var paymentResp PaymentResponse
    err = json.NewDecoder(resp.Body).Decode(&paymentResp)
    return paymentResp, err
}

// OrderWorkflow uses the activity
func OrderWorkflow(ctx workflow.Context, order Order) error {
    // Configure activity with retries
    activityOptions := workflow.ActivityOptions{
        StartToCloseTimeout: 60 * time.Second,
        RetryPolicy: &temporal.RetryPolicy{
            MaximumAttempts: 3,
        },
    }
    ctx = workflow.WithActivityOptions(ctx, activityOptions)

    // Call payment service via activity
    paymentReq := PaymentRequest{
        OrderID: order.ID,
        Amount:  order.Total,
    }

    var paymentResp PaymentResponse
    err := workflow.ExecuteActivity(ctx, CallPaymentServiceActivity, paymentReq).Get(ctx, &paymentResp)
    if err != nil {
        return err
    }

    // Continue with order processing...
    return nil
}
```

---

## Activity Wrapper: Synchronous vs Asynchronous

### Pattern A: Synchronous (Wait for Completion)

API endpoint waits for workflow to complete before returning.

**Pros:**
- Simple - one API call gets final result
- No polling needed

**Cons:**
- API must stay open for entire workflow duration
- Risk of timeout for long-running workflows
- Ties up connection resources

**When to use:** Short workflows (seconds to minutes)

---

### Pattern B: Asynchronous (Start + Poll)

API endpoint returns immediately with workflowID, caller polls for status.

```go
// Namespace B: Return workflowID immediately
func handlePaymentRequestAsync(w http.ResponseWriter, r *http.Request) {
    // ... parse request ...

    // Start workflow (don't wait)
    we, err := c.ExecuteWorkflow(context.Background(), workflowOptions, PaymentWorkflow, req)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    // Return immediately with workflowID
    response := PaymentResponse{
        WorkflowID: we.GetID(),
        RunID:      we.GetRunID(),
        Status:     "started",
    }
    json.NewEncoder(w).Encode(response)
}

// Separate endpoint to check status
func handlePaymentStatus(w http.ResponseWriter, r *http.Request) {
    workflowID := r.URL.Query().Get("workflow_id")

    // Query workflow for status
    c, _ := client.Dial(client.Options{...})
    defer c.Close()

    resp, err := c.QueryWorkflow(context.Background(), workflowID, "", "getStatus")
    // ... return status ...
}
```

```go
// Namespace A: Activity polls for completion
func CallPaymentServiceActivityAsync(ctx context.Context, req PaymentRequest) (PaymentResponse, error) {
    // 1. Start workflow
    startResp, err := startPaymentWorkflow(req)
    if err != nil {
        return PaymentResponse{}, err
    }

    // 2. Poll for completion
    for {
        status, err := checkPaymentStatus(startResp.WorkflowID)
        if err != nil {
            return PaymentResponse{}, err
        }

        if status.Status == "completed" {
            return status, nil
        }

        if status.Status == "failed" {
            return PaymentResponse{}, errors.New(status.Error)
        }

        // Wait before polling again
        time.Sleep(5 * time.Second)
    }
}
```

**Pros:**
- Works for long-running workflows
- No connection timeout issues
- API server doesn't hold state

**Cons:**
- More complex - need polling logic
- Multiple API calls
- Polling overhead

**When to use:** Long-running workflows (minutes to hours)

---

## Pros and Cons of Activity Wrapper

### ✅ Advantages

**1. Works Today**
- No special Temporal features required
- Standard HTTP/gRPC - well understood
- Works across any infrastructure (cloud, on-prem, different Temporal clusters)

**2. Full Control**
- You control the API design
- Can add custom authentication, rate limiting, logging
- Can expose subset of workflows (some workflows remain private)

**3. Flexible**
- Can aggregate multiple workflows behind single API
- Can add business logic in API layer
- Can version API independently

### ❌ Disadvantages

**1. Operational Burden[^1]**
- Must deploy and maintain API server
- Need load balancer, monitoring, alerting
- Another service to secure, scale, debug

**2. Some Boilerplate**
- Manual retry logic in activity
- Manual timeout handling
- Need to implement async polling pattern for long workflows

**3. Poor Observability**
- Workflow history shows "HTTP call" activity - not the actual downstream workflow
- Must manually correlate logs across namespaces
- No automatic linking between workflows

---

## Pattern 2: Temporal Nexus

TBD

---

## Additional Resources

- [Temporal Namespaces Best Practices](https://docs.temporal.io/best-practices/managing-namespace)

[^1]: If your service is already exposed through API, then adding a service to
    expose the workflow-related action is not a big overhead. Some companies
    actually have mandatory service must expose through API policies.
