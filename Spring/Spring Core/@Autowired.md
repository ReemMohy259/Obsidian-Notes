The *Spring Framework* uses `@Autowired` to inject dependent beans primarily by **matching the exact data type**. When multiple matching bean candidates exist, Spring resolves the ambiguity using a strict order of precedence: **`@Qualifier`**, **`@Primary`**, and finally **variable/field name matching**. 

#### How `@Autowired` Works Without Any Extra Annotations

- **By-Type Matching:** Spring searches its container for a bean that matches the requested field, setter, or constructor parameter type.

- **Single Match Success:** If it finds exactly one bean of that type, Spring injects it immediately.

- **Zero Matches:** If no bean of that type exists, Spring throws a runtime error unless you write `@Autowired(required = false)`.

- **Multiple Matches (Conflict):** If two or more beans of the exact same type exist (such as two classes implementing the same interface), Spring attempts bean name matching using the field or constructor parameter name. If that also fails to identify a unique bean, Spring throws a `NoUniqueBeanDefinitionException`.

---
#### How Resolution Works With Extra Annotations

When ambiguity happens, Spring checks these tools in order to pick the right bean:

- **`@Primary` (Default Choice):**
    - Placed on a specific class or bean definition method.
    - Tells Spring: _"If there are multiple choices for this type, pick this one by default."_
    - If only one bean has `@Primary`, Spring injects it and avoids errors.

- **`@Qualifier` (Specific Choice):**
    - Placed at the injection point (next to the field, constructor parameter, or setter parameter) along with a specific bean name string.
    - Tells Spring: _"Ignore any primary rules; inject the exact bean name I asked for."_
    - **Note:** `@Qualifier` overrides `@Primary` if both are present.

- **`@Priority` (Alternative Ordering):**
    - Part of the Jakarta/Java standard annotations (`@Priority`).
    - Works similarly to `@Primary` by setting a numerical priority value, but Spring natively leans toward `@Primary` or `@Qualifier` for fine-tuning bean wiring. If multiple beans have priority values, the lowest number wins, but it is less flexible than Spring's native annotations. 
- **Field Name Fallback (Last Resort):**
    - If no `@Primary`, `@Qualifier`, or `@Priority` is used, Spring looks at the local variable name where you are injecting the bean and compares it to available bean IDs.
    - For example, if you write `private Animal dog;`, Spring looks for a bean named `dog`. 

#### Summary
```
1. Find beans by TYPE

↓

If 0 → UnsatisfiedDependencyException

↓

If 1 → Inject

↓

If multiple →

    2. @Qualifier (explicit bean selection)

↓

    3. @Primary (default candidate)

↓

    4. Field/Constructor Parameter Name
       matches bean name

↓

If still ambiguous →

NoUniqueBeanDefinitionException
```