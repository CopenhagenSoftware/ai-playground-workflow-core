---
description: "Implement a new workflow operation (step primitive) with fluent builder support and full test coverage"
name: "New Workflow Operation"
argument-hint: "Describe the operation (e.g. 'RetryUntil — retries child steps until a condition is met')"
agent: agent
tools: [execute, read, edit, search, agent, todo]
---

Your task is to implement a new workflow operation (step primitive) in the WorkflowCore engine, following the established patterns exactly. The operation must include the runtime step body, fluent builder integration, and integration tests.

## Context

WorkflowCore is a .NET workflow engine. Operations are composed of:

1. **Primitive class** in [src/WorkflowCore/Primitives/](src/WorkflowCore/Primitives/) — the runtime step body
2. **Fluent builder methods** on [src/WorkflowCore/Services/FluentBuilders/StepBuilder.cs](src/WorkflowCore/Services/FluentBuilders/StepBuilder.cs) and/or [src/WorkflowCore/Services/FluentBuilders/WorkflowBuilder.cs](src/WorkflowCore/Services/FluentBuilders/WorkflowBuilder.cs) — wires the primitive into the builder DSL
3. **Builder interface additions** on [src/WorkflowCore/Interface/IStepBuilder.cs](src/WorkflowCore/Interface/IStepBuilder.cs) and/or [src/WorkflowCore/Interface/IWorkflowBuilder.cs](src/WorkflowCore/Interface/IWorkflowBuilder.cs)
4. **Integration test scenario** in [test/WorkflowCore.IntegrationTests/Scenarios/](test/WorkflowCore.IntegrationTests/Scenarios/)

Key base classes and interfaces:
- `StepBody` (sync) or `StepBodyAsync` (async) for simple steps
- `ContainerStepBody` for block/container operations (If, While, ForEach)
- `IStepExecutionContext` for runtime context
- `ExecutionResult` for controlling flow (`.Next()`, `.Branch()`, `.Persist()`, `.Sleep()`)
- `WorkflowStep<TStepBody>` for step metadata
- `MemberMapParameter` for input expression mapping
- `IContainerStepBuilder<TData, TStepBody, TReturnStep>` for operations with child blocks (`.Do(...)`)

## Steps

### 1. Understand the requirement

Read the user's description of the new operation. Classify it:
- **Simple step**: no children, executes once and moves on (like `Delay`, `WaitFor`)
- **Container/block step**: has child steps inside a `.Do(...)` block (like `While`, `If`, `ForEach`)
- **Decision step**: branches based on a value (like `Decide`)

### 2. Study an existing similar operation

Before writing any code, read the implementation of the most similar existing primitive:
- For containers: read [src/WorkflowCore/Primitives/While.cs](src/WorkflowCore/Primitives/While.cs) and its builder method on `StepBuilder.cs`
- For simple steps: read [src/WorkflowCore/Primitives/Delay.cs](src/WorkflowCore/Primitives/Delay.cs)
- For decision steps: read [src/WorkflowCore/Primitives/Decide.cs](src/WorkflowCore/Primitives/Decide.cs)

Also read the corresponding integration test (e.g. `WhileScenario.cs`, `DelayScenario.cs`, `DecisionScenario.cs`).

### 3. Implement the primitive class

Create a new file in `src/WorkflowCore/Primitives/`. Follow these rules:
- Inherit from the appropriate base class (`StepBody`, `StepBodyAsync`, or `ContainerStepBody`)
- Expose input properties for any parameters (these become the targets for `MemberMapParameter`)
- Use `ExecutionResult` to express the step's control flow behavior
- For container steps, manage `ControlPersistenceData` to track child branch lifecycle
- Keep the class minimal — no service dependencies unless absolutely necessary

### 4. Add fluent builder support

Add the builder method to **both** `StepBuilder.cs` and `WorkflowBuilder.cs` (check if both have the analogous method for similar operations). Also add the method signature to the corresponding interface (`IStepBuilder.cs` / `IWorkflowBuilder.cs` / `IWorkflowModifier.cs`).

Follow the established pattern:
```
1. Create WorkflowStep<TPrimitive>
2. Add MemberMapParameter for each input expression
3. WorkflowBuilder.AddStep(newStep)
4. Create StepBuilder<TData, TPrimitive>
5. Wire Step.Outcomes to connect to newStep
6. Return the appropriate builder type
```

For container operations, return `IContainerStepBuilder<...>` so `.Do(...)` is available.

### 5. Write integration tests

Create a new scenario file in `test/WorkflowCore.IntegrationTests/Scenarios/`. Follow the `WhileScenario.cs` pattern:

```csharp
public class XxxScenario : WorkflowTest<XxxScenario.XxxWorkflow, XxxScenario.MyDataClass>
{
    // Static counters/flags to verify execution

    // Nested step bodies (if needed)

    // Nested data class

    // Nested workflow definition using the new fluent method

    public XxxScenario() { Setup(); }

    [Fact]
    public void Scenario()
    {
        var workflowId = StartWorkflow(new MyDataClass());
        WaitForWorkflowToComplete(workflowId, TimeSpan.FromSeconds(30));

        // Assert execution counts, ordering, final data state
        GetStatus(workflowId).Should().Be(WorkflowStatus.Complete);
        UnhandledStepErrors.Count.Should().Be(0);
    }
}
```

Write at least these test cases:
- **Happy path**: the operation executes its intended behavior end-to-end
- **Edge case**: boundary condition (e.g., condition starts false, empty collection, zero iterations)
- **Data flow**: inputs and outputs map correctly through the operation

### 6. Build and test

Run these commands and fix any issues:

```
dotnet build src/WorkflowCore/WorkflowCore.csproj
dotnet build test/WorkflowCore.IntegrationTests/WorkflowCore.IntegrationTests.csproj
dotnet test test/WorkflowCore.IntegrationTests --filter "FullyQualifiedName~XxxScenario" --no-build
```

If unit tests exist for related executor behavior, also run:
```
dotnet test test/WorkflowCore.UnitTests --no-build
```

### 7. Verify no regressions

Run the full test suite to confirm nothing is broken:
```
dotnet test test/WorkflowCore.UnitTests --no-build
dotnet test test/WorkflowCore.IntegrationTests --no-build
```

## Rules

- Do NOT modify existing primitives or their tests unless the new operation requires extending a shared interface
- Match the coding style of adjacent files exactly (naming, spacing, access modifiers)
- Use `Expression<Func<TData, T>>` for builder method parameters that map to step inputs
- Every public method added to a builder must also be added to the corresponding interface
- Reset static test counters at the start of each test method if the test class has multiple `[Fact]` methods
- Do not add XML doc comments unless the existing codebase uses them in the same file
