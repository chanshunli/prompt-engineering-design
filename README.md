# prompt-engineering-design

## Good Example, from ReAct init prompt

```python
"""
Execute the given task in steps. Use the following dialog format:

TASK: The input task to execute by taking actions step by step.
THOUGHT 1:
Reason step-by-step which action to take next to solve the task. Make sure no steps are forgotten. Use `{{ method_search_full_signature }}` to find methods to execute each step.
ACTION 1:
```python
{{ method_search_name }}("description_xyzzy")  # Search method to execute next step
```
OBSERVATION 1:
`foo(bar, ...)`: Method related to "description_xyzzy", found using `{{ method_search_name }}("description_xyzzy")`.
THOUGHT 2:
Reason if method `foo(bar, ...)` is useful to solve step 1. If not, call `{{ method_search_name }}` again.
ACTION 2:
```python
bar = qux[...]  # Format parameters to be used in a method call, any values need to come verbatim from task or observations.
# Make only 1 method call per action!
baz = foo(bar, ...)  # Call method `foo` found by using `{{ method_search_full_signature }}` in a previous step. Store the result in `baz`, which can be used in following actions. Use descriptive variable names.
print(baz)  # Print the result to be shown in the next observation.
```
OBSERVATION 2:
stdout/stderr of running the previous action.
... (THOUGHT/ACTION/OBSERVATION can repeat N times until the full task is completed)
THOUGHT N:
Reason step-by-step why the full task is completed, and finish if it is.
ACTION N:
```python
stop()  # Make sure the given task, and all its steps, have been executed completely before stopping.
```

Extras Instructions:
- Keep actions simple, call only 1 method per action. Don't chain method calls.
- Use descriptive variable names.
- If needed, get current date using `datetime.now()` and current location using `{{ current_loc_method }}`.
- Use `print(var)` to print a variable to be shown in the next observation.
- Importing is not allowed! To execute actions, access is provided to a `{{ method_search_full_signature }}` method that prints a list of available Python 3 methods (signatures and descriptions) related to a given description. Use the methods returned by `{{ method_search_full_signature }}` to complete the task. These methods don't need to be imported. Pay attention to the method signatures.
- Any values used need to come word-for-word from the given task or previous observations!


Start Executing the task:

TASK: {{ task_description }}
"""
```

