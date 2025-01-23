I'll create a clear example to demonstrate the difference in parameter resolution between the "before" and "after" scenarios.

Let's consider this example TaskRun with StepAction:

```yaml
# TaskRun
apiVersion: tekton.dev/v1
kind: TaskRun
metadata:
  name: example-taskrun
spec:
  taskSpec:
    steps:
      - name: greet
        ref:
          name: greeting-action
        params:
          - name: message
            value: "Hi"

# StepAction
apiVersion: tekton.dev/v1beta1
kind: StepAction
metadata:
  name: greeting-action
spec:
  params:
    - name: message
      type: string
      default: "hello"
    - name: greeting
      type: string
      default: "$(params.message) world"
    - name: final-message
      type: string
      default: "$(params.greeting)!"
  image: ubuntu
  script: |
    echo "$(params.final-message)"
```

### Before the Changes:

The resolution would happen in a less structured way:

1. **Initial Parameter Loading**:
   ```
   message = "Hi"  # From step param
   greeting = "$(params.message) world"  # From default
   final-message = "$(params.greeting)!"  # From default
   ```

2. **Parameter Resolution** (single pass):
   ```
   # Replace all at once
   message = "Hi"
   greeting = "Hi world"  # Replaces $(params.message)
   final-message = "Hi world!"  # Replaces $(params.greeting)
   ```

Problems with this approach:
- No validation of parameter references
- Resolution order was not guaranteed
- Could miss some substitutions if dependencies weren't resolved in the right order
- No clear separation between different types of parameter substitutions

### After the Changes:

The resolution now follows a clear, ordered process:

1. **TaskRun Parameter Values** (none in this case)

2. **Step-provided Parameter Values**:
   ```
   message = "Hi"  # From step param
   ```

3. **Default Values with References** (ordered by dependency):
   First, identify parameter dependencies:
   ```
   greeting depends on: message
   final-message depends on: greeting
   ```
   
   Then resolve in order:
   ```
   # First resolve greeting (depends on message)
   greeting = "$(params.message) world"
   greeting = "Hi world"  # message is already resolved
   
   # Then resolve final-message (depends on greeting)
   final-message = "$(params.greeting)!"
   final-message = "Hi world!"  # greeting is now resolved
   ```

4. **Simple Default Values** (none in this case)

5. **Step Result References** (none in this case)

Let's look at a more complex example that shows the full power of the new system:

```yaml
# TaskRun
apiVersion: tekton.dev/v1
kind: TaskRun
metadata:
  name: complex-example
spec:
  params:
    - name: prefix
      value: "Welcome"
  taskSpec:
    params:
      - name: prefix
        type: string
    steps:
      - name: generate-message
        image: ubuntu
        script: |
          echo "friend" > $(results.recipient.path)
      - name: greet
        ref:
          name: greeting-action
        params:
          - name: custom-greeting
            value: "$(steps.generate-message.results.recipient)"

# StepAction
apiVersion: tekton.dev/v1beta1
kind: StepAction
metadata:
  name: greeting-action
spec:
  params:
    - name: prefix
      type: string
      default: "$(params.prefix)"
    - name: custom-greeting
      type: string
      default: "friend"
    - name: message
      type: string
      default: "$(params.prefix) $(params.custom-greeting)"
  image: ubuntu
  script: |
    echo "$(params.message)"
```

### After Changes Resolution Process:

1. **TaskRun Parameter Values**:
   ```
   prefix = "Welcome"  # From TaskRun params
   ```

2. **Step-provided Parameter Values**:
   ```
   custom-greeting = "$(steps.generate-message.results.recipient)"
   ```

3. **Default Values with References**:
   ```
   # First resolve prefix
   prefix = "$(params.prefix)" -> "Welcome"
   
   # message depends on both prefix and custom-greeting
   message = "$(params.prefix) $(params.custom-greeting)"
   ```

4. **Simple Default Values**:
   ```
   custom-greeting = "friend"  # Only used if step doesn't provide value
   ```

5. **Step Result References** (processed last):
   ```
   custom-greeting = "$(steps.generate-message.results.recipient)"
   message = "Welcome $(steps.generate-message.results.recipient)"
   ```

The key improvements in the new system are:
1. Clear ordering of substitutions
2. Proper handling of parameter dependencies
3. Separate processing of step results
4. Validation of parameter references
5. Prevention of circular dependencies

This makes the system more predictable and robust, especially when dealing with complex parameter relationships and step result references.

Before the changes, there were several key issues with step result references. Let me explain by analyzing the code and showing an example:

### Main Issues with Step Result References Before:

1. **Premature Resolution**: Step result references were being processed simultaneously with regular parameter substitution, rather than being handled last. This was problematic because:
   - Step results might not be available during parameter resolution
   - The ordering wasn't clear or predictable

2. **Incorrect Reference Handling**: The old code didn't properly distinguish between regular parameter values and step result references. Looking at the old code:

```go
// Old approach in replacementsFromStepResults
func replacementsFromStepResults(step *v1.Step, stepParams v1.Params, defaults []v1.ParamSpec) (map[string]string, error) {
    stringReplacements := map[string]string{}
    for _, sp := range stepParams {
        if sp.Value.StringVal != "" {
            //  $(params.p1) --> $(steps.step1.results.foo) (normal substitution)
            value := strings.TrimSuffix(strings.TrimPrefix(sp.Value.StringVal, "$("), ")")
            pr, err := resultref.ParseStepExpression(value)
            // ...
        }
    }
    // ...
}
```

3. **No Type-Specific Handling**: The old code didn't properly handle different parameter types (string, array, object) when dealing with step results.

Let's look at an example that would cause issues in the old system:

```yaml
apiVersion: tekton.dev/v1
kind: TaskRun
metadata:
  name: problematic-example
spec:
  taskSpec:
    steps:
      - name: step1
        image: ubuntu
        script: |
          echo "Hello" > $(results.greeting.path)
          echo '["a", "b", "c"]' > $(results.items.path)
      
      - name: step2
        ref:
          name: process-action
        params:
          - name: message
            value: "$(steps.step1.results.greeting)"
          - name: items
            value: "$(steps.step1.results.items)"
          - name: combined
            value: "$(params.message) with $(params.items[0])"

# StepAction
apiVersion: tekton.dev/v1beta1
kind: StepAction
metadata:
  name: process-action
spec:
  params:
    - name: message
      type: string
    - name: items
      type: array
    - name: combined
      type: string
      default: "$(params.message) with $(params.items[0])"
  image: ubuntu
  script: |
    echo "$(params.combined)"
```

### How it Would Break Before:

1. **Premature Resolution**:
```go
// The old system would try to resolve this immediately
combined = "$(params.message) with $(params.items[0])"
// Would try to substitute $(params.message) before step results were available
```

2. **Incorrect Array Handling**:
```yaml
items: "$(steps.step1.results.items)"
# Old system would treat this as a string, not recognizing it should be an array
```

3. **Mixed Parameter Resolution**:
```yaml
combined: "$(params.message) with $(params.items[0])"
# Would try to resolve regular params and step results at the same time
```

### How the New System Fixes These Issues:

1. **Ordered Resolution**:
```go
// New system processes in clear order:
// 1. Regular parameters
// 2. Default values
// 3. Step results last
stepResultReplacements, err := replacementsFromStepResults(step, stepParams, defaults)
// Merge step result replacements last
for k, v := range stepResultReplacements {
    stringReplacements[k] = v
}
```

2. **Better Type Handling**:
```go
// New system specifically checks for step result references
if sp.Value.StringVal != "" && strings.HasPrefix(sp.Value.StringVal, "$(steps.") {
    // Handle step result reference
}
```

3. **Clear Separation**:
```go
// Step results are now processed separately and last
if strings.HasPrefix(sp.Value.StringVal, "$(steps.") {
    // Process step result reference
    value := strings.TrimSuffix(strings.TrimPrefix(sp.Value.StringVal, "$("), ")")
    pr, err := resultref.ParseStepExpression(value)
    // ...
}
```

The new system ensures:
1. Step results are processed after all other parameter resolutions
2. Proper handling of array and object type results
3. Clear separation between regular parameter substitution and step result references
4. Correct ordering of substitutions to prevent premature resolutions
5. Better error handling for step result references

This makes the system much more reliable when dealing with step results, especially in complex scenarios where you're mixing regular parameters, default values, and step result references.

