# Camunda BPMN & DMN — Exercise 2

This repository contains the implementation of two **Camunda BPMN and DMN exercises** focused on business process automation, decision tables, rule-based routing, and dynamic task orchestration.

---

## Exercise 1 — Loan Application Risk Routing

### Problem Statement

A loan processing system needs to automatically evaluate the risk associated with a customer's loan application and route the application according to the calculated risk level.

The system receives the following loan application details:

- `applicantAge`
- `creditScore`
- `loanAmount`

A **Business Rule Task** invokes a DMN decision table to determine:

- `riskTier` — `LOW`, `MEDIUM`, or `HIGH`
- `requiresManualReview` — `true` or `false`

The resulting decision is then used by an **Exclusive Gateway (XOR)** to determine the next step in the loan approval process.

### BPMN Workflow

```text
Customer submits loan application
              |
              v
     Evaluate Loan Risk
       (Business Rule Task)
              |
              v
       Evaluate Risk
       (DMN Decision)
              |
              v
        XOR Gateway
        /     |     \
       v      v      v
     LOW    MEDIUM   HIGH
      |        |       |
      v        v       v
 Auto-      Underwriter Auto-Reject
 Approve      Review   Notification
 & Disburse
      |        |       |
      +--------+-------+
               |
          End Events
```

### Routing Rules

| Condition | Action |
|---|---|
| `riskTier == "LOW"` | Auto-Approve and Disburse |
| `riskTier == "MEDIUM"` OR `requiresManualReview == true` | Underwriter Review |
| `riskTier == "HIGH"` AND `requiresManualReview == false` | Auto-Reject Notification |

### DMN Decision Table

**Decision:** `evaluate-loan-risk`

**Hit Policy:** Unique (`U`) or First (`F`)

**Inputs:**
- `creditScore` — integer
- `loanAmount` — double

**Outputs:**
- `riskTier` — string
- `requiresManualReview` — boolean

| Rule | Credit Score | Loan Amount | Risk Tier | Manual Review |
|---|---|---|---|---|
| 1 | `>= 750` | `<= 50000` | `LOW` | `false` |
| 2 | `>= 750` | `> 50000` | `MEDIUM` | `true` |
| 3 | `[600..749]` | Any | `MEDIUM` | `true` |
| 4 | `< 600` | Any | `HIGH` | `false` |

### Implementation Requirements

- Create the BPMN 2.0 process in Camunda Modeler.
- Configure the Business Rule Task to execute the DMN decision.
- Configure the result variable mapping using **Single Result**.
- Implement the DMN decision table using appropriate FEEL unary tests.
- Configure the XOR Gateway's outgoing sequence-flow conditions using Camunda FEEL.

---

# Exercise 2 — Multi-Item Order Discount & Fulfillment Orchestration

### Problem Statement

An order processing system needs to calculate the total discount applicable to an order based on multiple independent conditions.

The process receives:

- `customerTier` — `REGULAR` or `PREMIUM`
- `cartValue` — double
- `promoCode` — string

A **Business Rule Task** invokes a DMN decision table. Unlike Exercise 1, multiple DMN rules can match at the same time, and their discount values must be **summed**.

The calculated discount is then used to determine the final invoice amount:

```text
finalAmount = cartValue * (1 - (totalDiscount / 100))
```

The final amount determines whether manager approval is required.

### BPMN Workflow

```text
Receive Order
     |
     v
Calculate Total Discount
   (Business Rule Task)
     |
     v
  DMN Decision
     |
     v
Calculate Final Amount
  (Script / Service Task)
     |
     v
   XOR Gateway
    /        \
   v          v
>= 1000     < 1000
   |          |
   v          |
Manager       |
Sign-off      |
   |          |
   +----+-----+
        |
        v
Send Order to Warehouse
        |
        v
       End
```

### Routing Rules

| Condition | Action |
|---|---|
| `finalAmount >= 1000` | Manager Sign-off |
| `finalAmount < 1000` | Proceed directly |
| Both paths | Send Order to Warehouse |

### DMN Decision Table

**Decision:** `calculate-discounts`

**Hit Policy:** Collect Sum (`C+`)

**Inputs:**
- `customerTier` — string
- `cartValue` — double
- `promoCode` — string

**Output:**
- `discountPercent` — integer

| Rule | Condition | Discount |
|---|---|---:|
| 1 | `customerTier == "PREMIUM"` | +10% |
| 2 | `cartValue >= 500` | +5% |
| 3 | `promoCode in ("FESTIVE10", "SPECIAL10")` | +10% |
| 4 | Default rule | +0% |

Because the DMN uses **Collect Sum (C+)**, multiple rules can apply concurrently and their outputs are aggregated into a single discount value.

### Implementation Requirements

- Configure the DMN table with **Collect Sum (C+)**.
- Define the appropriate output aggregator.
- Configure the Business Rule Task with **Single Entry** result mapping.
- Calculate the final invoice amount using the aggregated discount.
- Route the process through the XOR Gateway based on `finalAmount`.

### Test Case

The assignment specifies the following sample payload:

```json
{
  "customerTier": "PREMIUM",
  "cartValue": 600,
  "promoCode": "FESTIVE10"
}
```

Applicable discounts:

```text
Premium customer     = 10%
Cart value >= 500    =  5%
FESTIVE10 promo      = 10%
                         ----
Total discount       = 25%
```

Therefore:

```text
finalAmount = 600 × (1 - 25/100)
            = 600 × 0.75
            = 450
```

Since the final amount is **less than 1000**, the order proceeds directly to **Send Order to Warehouse** without Manager Sign-off.

---

## Technologies Used

- **Camunda**
- **BPMN 2.0**
- **DMN**
- **FEEL (Friendly Enough Expression Language)**
- **Camunda Modeler**

## Key Concepts Demonstrated

### Exercise 1
- Business Rule Task
- DMN risk evaluation
- Single-result mapping
- XOR Gateway
- FEEL conditions
- Risk-based process routing

### Exercise 2
- Multi-hit DMN evaluation
- Collect Sum (`C+`)
- Discount aggregation
- Single Entry result mapping
- Dynamic invoice calculation
- Conditional manager approval
- Order fulfillment orchestration

## Repository Structure

```text
camunda-loan-order-exercises/
|
├── Excercise 2/
│   |
│   ├── Ex 1/
│   │   ├── loan-risk-routing.bpmn
│   │   └── evaluate-loan-risk.dmn
│   │
│   └── Ex 2/
│       ├── assignment2.bpmn
│       └── assignment2.dmn
│
└── README.md
```

## How to Run

1. Open the BPMN and DMN files using **Camunda Modeler**.
2. Deploy the required BPMN and DMN definitions to the Camunda environment.
3. Start a process instance with the required input variables.
4. Verify the DMN decision result.
5. Verify that the BPMN process follows the expected gateway path.
6. For Exercise 2, verify that multiple applicable discounts are aggregated correctly.

## Summary

This project demonstrates how **BPMN handles process orchestration** while **DMN handles business decision logic**.

- **Exercise 1** focuses on **risk evaluation and routing a loan application** based on a single DMN decision.
- **Exercise 2** focuses on **aggregating multiple applicable discounts** and using the calculated final amount to dynamically route an order.

The two exercises demonstrate both **single-hit decision making** and **multi-hit decision aggregation** within Camunda.
