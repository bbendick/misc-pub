# Boomi Groovy Scripting Patterns

Reference for writing Groovy scripts in Boomi Data Process shapes and scripting steps.
Battle-tested patterns — do not deviate without good reason.

---

## Imports

Always import explicitly. Never rely on implicit availability.

```groovy
import com.boomi.execution.ExecutionUtil
import java.util.Properties
import java.io.InputStream
import java.io.ByteArrayInputStream
```

Add others as needed (`java.time.*`, `java.math.BigDecimal`, etc.).

---

## Logging

Use `ExecutionUtil.getBaseLogger()`. Never use `System.out.println()` for production scripts.

```groovy
import com.boomi.execution.ExecutionUtil

def logger = ExecutionUtil.getBaseLogger()
```

### Available log levels

| Method | Notes |
|---|---|
| `logger.info("msg")` | Standard logging — use this most |
| `logger.fine("msg")` | Verbose/debug — only visible at FINE log level |
| `logger.warning("msg")` | Warnings — note: NOT `.warn()`, that does not exist |
| `logger.severe("msg")` | Errors |

```groovy
logger.info("Processing started. Document count: " + dataContext.getDataCount())
logger.fine("Detailed debug value: " + someVar)
logger.warning("Could not parse value: '${rawValue}'")
```

> ⚠️ `logger.warn()` does **not** exist in Boomi's logger. Use `logger.warning()`.

---

## Reading Documents

Always use index-based access. Never use `nextDocument()` — it fails with Groovy's default Map type.

```groovy
for (int i = 0; i < dataContext.getDataCount(); i++) {
    InputStream is = dataContext.getStream(i)
    Properties props = dataContext.getProperties(i)

    // Read content
    String content = is.text.trim()

    // ... process content ...

    // Pass document through unchanged
    dataContext.storeStream(new ByteArrayInputStream(content.getBytes("UTF-8")), props)
}
```

### Why not `nextDocument()`?

Boomi expects a `java.util.Properties` object. Groovy's `[:]` creates a `Map`, not `Properties`, causing:
```
No signature of method: DataContextImpl.nextDocument() is applicable for argument types: (java.util.Properties) values: [[:]]]
```

`getStream(i)` / `getProperties(i)` sidesteps this entirely.

---

## Writing Documents

Always use `ByteArrayInputStream` with explicit UTF-8 encoding.

```groovy
String outputContent = "your content here"
InputStream outputStream = new ByteArrayInputStream(outputContent.getBytes("UTF-8"))
dataContext.storeStream(outputStream, props)
```

### Writing a new document (not passing through)

```groovy
Properties outProps = new Properties()
// optionally set properties on outProps here
InputStream outputStream = new ByteArrayInputStream("content".getBytes("UTF-8"))
dataContext.storeStream(outputStream, outProps)
```

> ⚠️ Even though `ByteArrayInputStream` is technically an `InputStream` (not an OutputStream), the variable name should reflect its use in context. Name it `outputStream` if it's what you're writing out; just be aware the type is `InputStream`.

---

## Dynamic Process Properties (DPP)

Process-scoped. Read and write via `ExecutionUtil`.

### Reading

```groovy
String myValue = ExecutionUtil.getDynamicProcessProperty("dpp_my_property")
```

### Writing

```groovy
ExecutionUtil.setDynamicProcessProperty("dpp_my_property", "someValue", false)
```

The third parameter (`false`) controls whether to persist the property to child processes. Use `false` unless you have a specific reason to propagate.

### Convention

Prefix all DPP names with `dpp_` for clarity:
```groovy
ExecutionUtil.getDynamicProcessProperty("dpp_ldap_user")
ExecutionUtil.getDynamicProcessProperty("dpp_date_format")
ExecutionUtil.setDynamicProcessProperty("dpp_output_date", formattedDate, false)
```

---

## Dynamic Document Properties (DDP)

Document-scoped. Read and write via the `props` object for that document.

### Reading

```groovy
String val = props.getProperty("document.dynamic.userdefined.ddp_my_property")
```

### Checking existence

```groovy
// Truthy check (null or empty = false)
if (props.getProperty("document.dynamic.userdefined.ddp_my_property")) { ... }

// Explicit null check
if (props.getProperty("document.dynamic.userdefined.ddp_my_property") != null) { ... }

// Key existence (value could still be empty)
if (props.containsKey("document.dynamic.userdefined.ddp_my_property")) { ... }
```

### Writing

Set the property on `props` **before** calling `storeStream`. The property travels with the document.

```groovy
props.setProperty("document.dynamic.userdefined.ddp_destination", "GL")
dataContext.storeStream(outputStream, props)
```

### Convention

Prefix all DDP names with `ddp_`:
```groovy
props.setProperty("document.dynamic.userdefined.ddp_destination", destination)
props.setProperty("document.dynamic.userdefined.ddp_has_balance", "true")
```

---

## Base Document Properties

Boomi built-in document properties (not user-defined) use a different key prefix:

```groovy
props.setProperty("document.base.ApplicationStatusCode", "200")
```

---

## Full Script Template

Standard pattern for a data process shape script:

```groovy
import com.boomi.execution.ExecutionUtil
import java.util.Properties
import java.io.InputStream
import java.io.ByteArrayInputStream

def logger = ExecutionUtil.getBaseLogger()

logger.info("Script started. Document count: " + dataContext.getDataCount())

for (int i = 0; i < dataContext.getDataCount(); i++) {
    InputStream is = dataContext.getStream(i)
    Properties props = dataContext.getProperties(i)

    String content = is.text.trim()
    logger.info("Document ${i}: ${content.length()} chars")

    // --- your logic here ---

    // Write result back (modify content or props as needed)
    dataContext.storeStream(
        new ByteArrayInputStream(content.getBytes("UTF-8")),
        props
    )
}

logger.info("Script complete")
```

---

## Pass-Through (No Content Change)

When you only need to read/set properties but not modify the document body:

```groovy
for (int i = 0; i < dataContext.getDataCount(); i++) {
    InputStream is = dataContext.getStream(i)
    Properties props = dataContext.getProperties(i)

    // set a DDP
    props.setProperty("document.dynamic.userdefined.ddp_processed", "true")

    dataContext.storeStream(is, props)
}
```

---

## DPP-Only Script (No Documents)

When the script only needs to set process properties and pass documents through unchanged:

```groovy
import com.boomi.execution.ExecutionUtil
import java.util.Properties
import java.io.InputStream

def logger = ExecutionUtil.getBaseLogger()

// Do your DPP work
String inputValue = ExecutionUtil.getDynamicProcessProperty("dpp_input")
ExecutionUtil.setDynamicProcessProperty("dpp_output", inputValue.toUpperCase(), false)

// Always pass documents through
for (int i = 0; i < dataContext.getDataCount(); i++) {
    InputStream is = dataContext.getStream(i)
    Properties props = dataContext.getProperties(i)
    dataContext.storeStream(is, props)
}
```

---

## Dual-Mode Scripts (Local + Boomi)

For scripts that need to run both locally (for development/testing) and inside Boomi, use `binding.hasVariable('dataContext')` to detect the environment and branch only the I/O. All business logic stays identical.

### Key rules

- Do **not** `import com.boomi.execution.ExecutionUtil` at the top — it won't resolve locally. Use the fully qualified name **inside the Boomi branch only**.
- Standard Java imports (`java.util.Properties`, `java.io.ByteArrayInputStream`) are safe at the top — they exist in both environments.
- `@Field` variables and all functions are shared between both branches.

### Pattern

```groovy
import java.time.LocalDate
import groovy.json.JsonOutput
import groovy.transform.Field
import java.util.Properties
import java.io.ByteArrayInputStream

// @Field constants and all business logic functions defined here...
// No references to ExecutionUtil or dataContext at this level.

// ============ ENTRY POINT ============
if (binding.hasVariable('dataContext')) {
    // ---- Boomi ----
    def logger = com.boomi.execution.ExecutionUtil.getBaseLogger()
    logger.info("Script started. Document count: " + dataContext.getDataCount())

    for (int i = 0; i < dataContext.getDataCount(); i++) {
        InputStream is   = dataContext.getStream(i)
        Properties props = dataContext.getProperties(i)

        String content = is.getText('ISO-8859-1')

        // Run business logic
        def result = processData(content)

        // Output one or more documents, tagged with DDPs for downstream routing
        Properties outProps = new Properties()
        outProps.setProperty("document.dynamic.userdefined.ddp_destination", "GL")
        dataContext.storeStream(new ByteArrayInputStream(JsonOutput.toJson(result).getBytes("UTF-8")), outProps)

        logger.info("Done. Output rows: " + result.size())
    }

} else {
    // ---- Local ----
    def result = processData(new File("in/input.txt").getText('ISO-8859-1'))

    new File("out").mkdirs()
    new File("out/result.json").text = JsonOutput.prettyPrint(JsonOutput.toJson(result))
    println "Output rows: ${result.size()} -> out/result.json"
}
```

### Why not import ExecutionUtil at the top?

Groovy resolves class references lazily at runtime, not at compile time. Placing `ExecutionUtil` usage only inside the `if (isBoomi)` branch means it is never resolved when running locally. A top-level `import` would force resolution and fail locally if the class is not on the classpath.

### Multiple output documents

When a single input document should produce multiple output types (e.g. GL, AP_CN, AP_CA), write each as a separate document tagged with a DDP. Downstream Boomi routing shapes can then split by `ddp_destination`.

```groovy
Properties glProps = new Properties()
glProps.setProperty("document.dynamic.userdefined.ddp_destination", "GL")
dataContext.storeStream(new ByteArrayInputStream(JsonOutput.toJson(gl).getBytes("UTF-8")), glProps)

Properties cnProps = new Properties()
cnProps.setProperty("document.dynamic.userdefined.ddp_destination", "AP_CN")
dataContext.storeStream(new ByteArrayInputStream(JsonOutput.toJson(apCN).getBytes("UTF-8")), cnProps)
```

---

## Groovy Version Compatibility

Boomi runs **Groovy 2.4**. Many methods added in Groovy 3+ will compile and run fine locally (e.g. with Groovy 4) but fail silently or throw at runtime in Boomi. The error is usually a `MissingMethodException` that only surfaces in Boomi.

### Known Groovy 3+ methods unavailable in Boomi (2.4)

| Avoid (3+) | Use instead (2.4-safe) |
|---|---|
| `str.takeRight(n)` | `str.with { it.length() > n ? it.substring(it.length() - n) : it }` |
| `str.stripLeading()` / `str.stripTrailing()` | `str.replaceAll(/^\s+/, '')` / `str.replaceAll(/\s+$/, '')` |
| `List.copyOf(...)` | `new ArrayList<>(list)` |
| `Map.copyOf(...)` | `new LinkedHashMap<>(map)` |
| `str.isBlank()` | `str == null \|\| str.trim().isEmpty()` |

> ⚠️ Groovy 4 silently resolves some of these via extension methods on the local JVM, so local tests pass. Always validate new string/collection methods against the 2.4 docs before using in Boomi.

---

## Common Pitfalls

| Mistake | Correct |
|---|---|
| `logger.warn("msg")` | `logger.warning("msg")` |
| `def props = [:]` passed to `nextDocument(props)` | Use `getStream(i)` / `getProperties(i)` |
| `import com.boomi.execution.ExecutionUtil` missing | Always import it explicitly |
| Setting DDP after `storeStream` | Set it **before** `storeStream` |
| Using `.bytes` for encoding | Use `.getBytes("UTF-8")` explicitly |
| `execution.getDynamicProcessProperty()` | `ExecutionUtil.getDynamicProcessProperty()` |
| `str.takeRight(n)` (Groovy 3+) | `str.with { it.length() > n ? it.substring(it.length() - n) : it }` |
