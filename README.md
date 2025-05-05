# iCKB/Solver

A Linear Solver, a barebones fork of [Ivordir/YALPS](https://github.com/Ivordir/YALPS).

## YALPS Quick Guide

This guide covers the usage of YALPS, a lightweight, performant linear programming solver, for solving linear optimization problems. YALPS models LP problems using a column-wise approach. Model can be defined using object or iterable formats.

## Real-World Examples

### Cutting Stock Testcase in Object Format

This example illustrates minimizing costs while ensuring a minimum length constraint for stock.

```typescript
const cuttingStockModel = {
  direction: "minimize",  // Alternatively  direction: "maximize",
  objective: "cost",
  variables: {
    "21": { length: 21, cost: 75 },
    "26": { length: 26, cost: 90 },
    "31": { length: 31, cost: 105 },
    "36": { length: 36, cost: 120 }
  },
  constraints: {
    length: { min: 25 } // Alternatively `{ max: value }`, `{ equal: value }` or `{ min: lower, max: upper }`
  },
  integers: ["21", "26", "31", "36"], // Integer variables. If all variables are integers, it can be written as `integers: true`.
  binaries: [] // Binaries variables follow the very same rules as integers.
};

const cuttingStockSolution = solve(cuttingStockModel);
// Expected result: { status: "optimal", result: 90, variables: [ ["26", 1] ] }
```

### Berlin Air Lift Testcase with Iterable Format and Helper Functions

YALPS provides helper functions for generating constraint objects:

- `lessEq(value: number)` – creates a `{ max: value }` constraint.
- `greaterEq(value: number)` – creates a `{ min: value }` constraint.
- `equalTo(value: number)` – creates a `{ equal: value }` constraint.
- `inRange(lower: number, upper: number)` – creates a constraint with both `{ min: lower, max: upper }`.

This example demonstrates maximizing capacity while adhering to constraints on planes, persons, costs, and a specific condition for "yankees."

```typescript
// Define constraints using helper functions
const constraints = new Map<string, Constraint>()
  .set("plane", lessEq(44))    // Using lessEq for maximum plane constraint
  .set("person", lessEq(512))   // Using lessEq for maximum person constraint
  .set("cost", lessEq(300))     // Using lessEq for maximum cost constraint
  .set("yankees", equalTo(0));  // Using helper function for equality

// Define variables using object and iterable types
const brit = new Map<string, number>()
  .set("capacity", 20000)
  .set("plane", 1)
  .set("person", 8)
  .set("cost", 5)
  .set("yankees", -2);

const yank = new Map<string, number>()
  .set("capacity", 30000)
  .set("plane", 1)
  .set("person", 16)
  .set("cost", 9)
  .set("yankees", 1);

const berlinModel = {
  direction: "maximize",
  objective: "capacity",
  constraints,
  variables: {
    brit,
    yank
  }
};

const berlinSolution = solve(berlinModel);
// Expected result: { status: "optimal", result: 1024000, variables: [ ["brit", 12.8], ["yank", 25.6] ] }
```

## Advanced Options

For complex problems, configure advanced options by passing an options object to the `solve` function:

```typescript
const options = {
  precision: 1E-8,     // Treat numbers <= this value as zero
  checkCycles: false,  // Enable explicit cycle checking if needed
  maxPivots: 8192,     // Maximum number of simplex pivots
  tolerance: 0.05,     // Tolerance for returning near-optimal integer solutions
  timeout: 1000,       // Maximum time in milliseconds for the solve
  maxIterations: 32768,// Maximum iterations for branch and cut
  includeZeroVariables: false // Whether to list variables with 0 value in the solution
};

const solutionWithOptions = solve(model, options);
```

## Dependencies

```mermaid
graph TD;
    A["@ickb/solver"] --> B["@ickb/utils"];

    click A "https://github.com/ickb/solver" "Go to @ickb/solver"
    click B "https://github.com/ickb/utils" "Go to @ickb/utils"
```

## Epoch Semantic Versioning

This repository follows [Epoch Semantic Versioning](https://antfu.me/posts/epoch-semver). In short ESV aims to provide a more nuanced and effective way to communicate software changes, allowing for better user understanding and smoother upgrades.

## Licensing

This source code, crafted with care by [Ivordir](https://github.com/Ivordir) and slightly modified by [Phroi](https://phroi.com/), is freely available on [GitHub](https://github.com/ickb/solver) and it is released under the [MIT License](./LICENSE).