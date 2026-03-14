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

## Common Pitfalls

| Mistake | Correct |
|---|---|
| `logger.warn("msg")` | `logger.warning("msg")` |
| `def props = [:]` passed to `nextDocument(props)` | Use `getStream(i)` / `getProperties(i)` |
| `import com.boomi.execution.ExecutionUtil` missing | Always import it explicitly |
| Setting DDP after `storeStream` | Set it **before** `storeStream` |
| Using `.bytes` for encoding | Use `.getBytes("UTF-8")` explicitly |
| `execution.getDynamicProcessProperty()` | `ExecutionUtil.getDynamicProcessProperty()` |
