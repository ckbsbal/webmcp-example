# WebMCP Store — Developer Reference

A fully working reference implementation of [WebMCP](https://github.com/GoogleChromeLabs/webmcp-tools), Google Chrome's proposed web standard for exposing structured tools to in-browser AI agents. This demo shows both the **Imperative API** (JavaScript) and the **Declarative API** (HTML form annotations) working together in a real e-commerce UI.

---

## Table of Contents

1. [What is WebMCP?](#what-is-webmcp)
2. [How it Works](#how-it-works)
3. [Core API Reference](#core-api-reference)
   - [Imperative API](#imperative-api)
   - [Declarative API](#declarative-api)
   - [Agent Lifecycle Events](#agent-lifecycle-events)
   - [SubmitEvent Extensions](#submitevent-extensions)
4. [Tools in This Demo](#tools-in-this-demo)
5. [Setup Guide](#setup-guide)
6. [Using the Model Context Tool Inspector](#using-the-model-context-tool-inspector)
7. [Project Structure](#project-structure)
8. [Best Practices](#best-practices)

---

## What is WebMCP?

WebMCP is a proposed web standard that allows websites to **publish structured tools** directly to in-browser AI agents. Instead of an agent guessing how to interact with a page by reading pixels or scraping DOM text, the website explicitly declares what it can do, what parameters each action requires, and what it returns.

Think of it as an API contract between your web app and any AI agent running in the browser.

**Without WebMCP**, an agent trying to search for a product must:
- Identify the search box visually
- Click into it, hoping it found the right element
- Type text character by character
- Find and click the search button
- Parse the results from rendered HTML

**With WebMCP**, the agent calls `search_products({ query: "iPhone" })` and gets back a structured result instantly. The website handles its own UI update.

### WebMCP vs MCP (Model Context Protocol)

| | MCP | WebMCP |
|---|---|---|
| **Where tools run** | Server-side | Client-side, in the browser tab |
| **Who deploys them** | Developers run a separate MCP server | Tools are embedded in the existing web page |
| **Transport** | JSON-RPC over stdio or HTTP | `window.navigator.modelContext` JS API |
| **Best for** | Desktop apps, CLI tools, backend APIs | Web apps and sites |
| **Requires a server** | Yes | No |

WebMCP is inspired by MCP but solves a different problem: giving agents native access to the web page they are already looking at.

---

## How it Works

WebMCP exposes a JavaScript API at `window.navigator.modelContext`. Your page registers tools on this object. A browser agent (or the Model Context Tool Inspector extension) can then discover and call those tools.

There are two ways to register tools:

**Imperative** — you write JavaScript that defines the tool name, schema, and an `execute` function. Full control, good for dynamic tools that depend on runtime state.

**Declarative** — you annotate an existing HTML `<form>` with special attributes (`toolname`, `tooldescription`, etc.). The browser automatically generates the tool definition from the form's structure. Good for forms that already exist.

Both approaches coexist in the same page and appear together in the tool inspector.

---

## Core API Reference

### Imperative API

#### `navigator.modelContext.registerTool(definition)`

Registers a single tool without touching any other registered tools. **Use this when declarative tools are also present** — the alternative `provideContext()` replaces the entire tool set and would wipe any browser-registered declarative tools.

```javascript
navigator.modelContext.registerTool({
  name: "search_products",

  // Plain-language description used by the model to decide when to call this tool.
  // Be specific about what it does and when to use it. Avoid negatives.
  description: "Search the product catalog by keyword. Use this when the user asks to find, browse, or filter products.",

  // JSON Schema describing the tool's input parameters
  inputSchema: {
    type: "object",
    properties: {
      query: {
        type: "string",
        description: "Keyword to search for in product names."
      }
    },
    required: ["query"]
  },

  // Optional hints to the agent about the tool's behaviour
  annotations: {
    readOnlyHint: "true"   // Does not modify any state — safe to call freely
  },

  // Called by the agent (or the inspector) with the parsed arguments
  execute: ({ query }) => {
    const results = filterProducts(query);
    renderProducts(results);

    // Return value is sent back to the agent as the tool's output
    return {
      content: [{
        type: "text",
        text: `Found ${results.length} product(s): ${results.map(p => p.name).join(", ")}`
      }]
    };
  }
});
```

**Key rules for `execute`:**
- Always return `{ content: [{ type: "text", text: "..." }] }`
- Return descriptive errors when inputs are invalid — the model uses them to self-correct and retry
- Update the UI before returning so the agent sees a consistent page state

---

#### `navigator.modelContext.provideContext({ tools })`

Registers multiple tools at once but **replaces the entire existing tool set**. Only use this when you want to completely reset all tools, for example when the app navigates to a fundamentally different state. Never use it alongside declarative form tools — it will silently remove them.

```javascript
navigator.modelContext.provideContext({
  tools: [toolA, toolB, toolC]
});
```

---

#### `navigator.modelContext.unregisterTool(name)`

Removes a specific tool by name without affecting others. Useful for removing tools that only apply in certain UI states, for example removing `checkout_order` once an order has already been placed.

```javascript
navigator.modelContext.unregisterTool("checkout_order");
```

---

#### `navigator.modelContext.clearContext()`

Removes all registered tools at once. Use when the user navigates to a part of the app where none of the current tools are relevant.

```javascript
navigator.modelContext.clearContext();
```

---

### Declarative API

The declarative API converts standard HTML forms into WebMCP tools automatically. You add special attributes to your `<form>` and its fields — the browser generates the tool definition from the markup.

#### Form-level attributes

| Attribute | Required | Description |
|---|---|---|
| `toolname` | Yes | The tool's identifier (e.g. `checkout_order`). Must be unique on the page. |
| `tooldescription` | Yes | Plain-language description for the model. Be specific and action-oriented. |
| `toolautosubmit` | No | When present, the browser submits the form automatically after the agent fills it. Without it, the browser pre-fills fields but waits for the user to click Submit. |

#### Field-level attributes

| Attribute | Applies to | Description |
|---|---|---|
| `toolparamtitle` | Any input | Overrides the JSON schema property key. Defaults to the field's `name` attribute. |
| `toolparamdescription` | Any input | Adds a description to the field's schema property. Defaults to the associated `<label>` text content. |

#### Example

```html
<form
  toolname="checkout_order"
  tooldescription="Place an order using the current cart. Requires name, email, and shipping address."
  toolautosubmit
>
  <input
    name="address"
    toolparamtitle="streetAddress"
    toolparamdescription="Full street address including apartment number"
    required
  />

  <select name="country" toolparamdescription="Destination country for delivery" required>
    <option value="India">India</option>
    <option value="USA">USA</option>
  </select>

  <button type="submit">Place Order</button>
</form>
```

The browser generates this JSON schema from the markup automatically:

```json
{
  "name": "checkout_order",
  "description": "Place an order using the current cart...",
  "inputSchema": {
    "type": "object",
    "properties": {
      "streetAddress": {
        "type": "string",
        "description": "Full street address including apartment number"
      },
      "country": {
        "type": "string",
        "enum": ["India", "USA"],
        "description": "Destination country for delivery"
      }
    },
    "required": ["country"]
  }
}
```

---

### Agent Lifecycle Events

These events fire on `window` and let you synchronise your UI with agent activity.

#### `toolactivated`

Fires when an agent has pre-filled a declarative form's fields and is ready to proceed. Use this to ensure the form is visible — for example, opening a modal that contains the form.

```javascript
window.addEventListener('toolactivated', ({ toolName }) => {
  if (toolName === "checkout_order") {
    openCheckoutModal();
    showAgentBanner("Agent is filling your shipping details…");
  }
});
```

#### `toolcancel`

Fires when the user cancels the agentic operation or `form.reset()` is called. Use this to restore the UI to its pre-agent state.

```javascript
window.addEventListener('toolcancel', ({ toolName }) => {
  if (toolName === "checkout_order") {
    closeCheckoutModal();
    hideAgentBanner();
  }
});
```

Both events carry a `toolName` string property matching the form's `toolname` attribute.

---

### SubmitEvent Extensions

WebMCP extends the standard `SubmitEvent` with two new members, available inside form `submit` event listeners.

#### `event.agentInvoked` — `boolean`

`true` when the form was submitted by an AI agent rather than a human click. Use this to branch your submit handler: return machine-readable data to the agent, or show normal UI feedback to the user.

```javascript
document.getElementById("checkoutForm").addEventListener("submit", function(e) {
  e.preventDefault();

  const order = processOrder(Object.fromEntries(new FormData(this)));

  if (e.agentInvoked) {
    // Return structured data — the agent uses this in follow-up steps
    e.respondWith(Promise.resolve({
      status: "success",
      orderId: order.id,
      total: order.total
    }));
  } else {
    // Normal human submission — show the success modal
    showSuccessModal(order);
  }
});
```

#### `event.respondWith(promise)` — `void`

Passes a promise to the browser that resolves with the tool's output. The resolved value is serialised and returned to the model. You **must** call `e.preventDefault()` before calling `respondWith`.

Always return a structured object rather than a plain string — the agent can then use specific fields (like `orderId`) in follow-up actions without parsing prose.

```javascript
// ✅ Good — structured and machine-readable
e.respondWith(Promise.resolve({
  status: "success",
  orderId: "ORD-001234",
  itemCount: 3,
  message: "Order placed for Jane Doe."
}));

// ❌ Avoid — unstructured strings are harder for agents to act on
e.respondWith(Promise.resolve("Order placed successfully"));
```

If validation fails, return an error object so the agent can self-correct:

```javascript
if (cart.length === 0) {
  e.respondWith(Promise.resolve({
    status: "error",
    message: "Cart is empty. Call add_to_cart before checkout_order."
  }));
  return;
}
```



---

## Tools in This Demo

This demo registers three tools using a mix of both APIs.

### `search_products` — Imperative

Filters and renders products matching a keyword. Marked with `readOnlyHint` because it does not modify cart or application state.

```
Input:  { query: string }
Output: { content: [{ type: "text", text: "Found 2 product(s): ..." }] }
```

### `add_to_cart` — Imperative

Adds a product to the cart by ID and quantity. Returns a descriptive error if the product ID is invalid, with the full list of valid IDs so the agent can self-correct and retry.

```
Input:  { productId: string, quantity: number }
Output: { content: [{ type: "text", text: "Added 1x iPhone 15 to cart. Total: $800." }] }
```

### `checkout_order` — Declarative

Registered automatically by the browser from the checkout `<form>` element. The agent pre-fills the shipping fields; the `toolactivated` event opens the checkout modal so the user can review before confirming.

```
Input:  { fullName, email, streetAddress, city, postalCode, country }
Output: { status, orderId, itemCount, total, message }  (via respondWith)
```

---

## Setup Guide

### Prerequisites

| Requirement | Details |
|---|---|
| Chrome Beta | Version 146 or higher |
| WebMCP flag | Must be enabled in `chrome://flags` |
| Model Context Tool Inspector | Chrome extension |
| Local web server | Any static file server |

---

### Step 1 — Install Chrome Beta

WebMCP is only available in **Chrome Beta version 146 or higher**. It is not in stable Chrome, Firefox, or Safari.

Download Chrome Beta from the official page:

```
https://www.google.com/chrome/beta/
```

Chrome Beta installs alongside your existing Chrome and does not replace it. You can run both at the same time.

---

### Step 2 — Enable the WebMCP Flag

1. Open **Chrome Beta** (not regular Chrome)
2. In the address bar, navigate to:

```
chrome://flags/#enable-webmcp-testing
```

3. Find the **"WebMCP for testing"** flag at the top of the page
4. Change the dropdown from **Default** to **Enabled**
5. Click the **Relaunch** button that appears at the bottom of the screen

Chrome Beta restarts with WebMCP active. This setting persists — you only need to do it once.

> **Verify it worked** — open DevTools console and run `typeof navigator.modelContext`. It should print `"object"`. If it prints `"undefined"`, the flag is not enabled or you are using regular Chrome instead of Chrome Beta.

---

### Step 3 — Install the Model Context Tool Inspector Extension

1. Open **Chrome Beta**
2. Go to the Chrome Web Store and search for:

```
WebMCP - Model Context Tool Inspector
```

3. Click **Add to Chrome**, then confirm the install
4. Pin the extension to your toolbar for easy access: click the puzzle piece icon (Extensions) → find the extension → click the pin icon

---

### Step 4 — Serve the Demo Locally

The demo is a single HTML file. It must be served over HTTP — you cannot open it as a `file://` URL because certain browser security policies block some APIs on file origins.

**Option A — Python (no install required)**

```bash
# Navigate to the folder containing index.html
cd /path/to/webmcp-store

# Python 3
python3 -m http.server 5500

# Python 2
python -m SimpleHTTPServer 5500
```

**Option B — Node.js**

```bash
npx serve . -p 5500
# or
npx http-server . -p 5500
```

**Option C — VS Code Live Server**

If you use VS Code, install the **Live Server** extension and click "Go Live" in the status bar. It opens the page in your default browser — make sure that browser is Chrome Beta.

Then open the demo in Chrome Beta:

```
http://127.0.0.1:5500/index.html
```

---

### Step 5 — Confirm Everything is Working

Open the browser console (F12 → Console tab) and run:

```javascript
// Should print: "object"
console.log(typeof navigator.modelContext);

// Should list registered tools
navigator.modelContextTesting.getTools().then(console.log);
```

If `navigator.modelContext` is `undefined`, check:
- You are using **Chrome Beta**, not regular Chrome
- The flag at `chrome://flags/#enable-webmcp-testing` is set to **Enabled**
- You clicked **Relaunch** after enabling the flag

---

## Using the Model Context Tool Inspector

The extension gives you three capabilities: listing registered tools, calling them manually with custom arguments, and driving an AI agent via Gemini.

### Listing Registered Tools

1. Open the demo at `http://127.0.0.1:5500/index.html` in Chrome Beta
2. Click the **Model Context Tool Inspector** icon in the toolbar
3. The panel opens and lists all tools registered on the current page

You should see all three tools: `search_products`, `add_to_cart`, and `checkout_order`. For each one you can see the name, description, and the full JSON input schema that would be sent to an AI model.

> If tools do not appear, check that the flag is enabled and that you relaunched Chrome Beta. For `checkout_order` specifically, the form must be present in the DOM — open the checkout modal first if needed, then refresh the extension panel.

---

### Calling a Tool Manually

Manual execution bypasses the AI entirely — you provide the arguments yourself. This is the fastest way to test whether your tool implementation is correct.

1. Select a tool from the **Tool** dropdown
2. Enter arguments as a JSON object in the **Input Arguments** field

For `add_to_cart`:
```json
{
  "productId": "iphone-15",
  "quantity": 2
}
```

For `search_products`:
```json
{
  "query": "laptop"
}
```

For `checkout_order`:
```json
{
  "fullName": "Jane Doe",
  "email": "jane@example.com",
  "streetAddress": "123 Main St",
  "city": "Mumbai",
  "postalCode": "400001",
  "country": "India"
}
```

3. Click **Execute Tool**
4. The extension calls your tool's `execute` function (or submits the form, for declarative tools) and displays the return value
5. The page UI updates in real time

If execution fails, the extension shows the error message from your `execute` function and links to the DevTools stack trace.

---

### Prompting the Agent with Natural Language

This tests the complete agent loop: your natural language → Gemini selects the right tool → Gemini provides arguments → tool executes → result returned to Gemini.

1. Click **Set a Gemini API Key** in the extension panel
2. Enter your API key (free at [aistudio.google.com](https://aistudio.google.com))
3. Type a natural language prompt in the **User Prompt** field:

```
Search for phones, add one Google Pixel to my cart, then take me to checkout 
with the name Alex Smith, email alex@example.com, address 42 Elm Street, 
city Bangalore, postal code 560001, country India.
```

4. Click **Send**
5. The extension sends your message to **Gemini 2.5 Flash** along with all registered tool schemas
6. Gemini decides which tools to call and in what order
7. The extension executes each tool call and shows the arguments and results

This is how you validate whether your tool descriptions and schemas are clear enough for a real model to use reliably. If Gemini picks the wrong tool or provides incorrect parameters, refine the `description` field on the tool and on individual parameters.

---

### Debugging Tips

| Problem | What to check |
|---|---|
| No tools appear in the extension | Verify `navigator.modelContext` is not `undefined` in the console. Ensure the flag is enabled and Chrome Beta was relaunched. |
| Declarative `checkout_order` is missing | The form must be in the DOM when the inspector loads. Open the checkout modal before opening the extension panel. |
| Agent picks the wrong tool | Improve the `description` field. Add examples. Clarify when to use each tool and what distinguishes them. |
| Agent sends wrong parameter values | Add examples and valid value lists to the parameter `description` fields. |
| `execute` throws an error | Click the error in the extension to open the DevTools stack trace. |
| `respondWith` is not working | Confirm you called `e.preventDefault()` before `e.respondWith()`. |

---

## Project Structure

```
index.html
│
├── Imperative Tools  (navigator.modelContext.registerTool)
│   ├── search_products     filters product grid · readOnlyHint: true
│   ├── add_to_cart         updates cart state and re-renders sidebar
│   └── confirm_order       programmatically submits the checkout form
│
├── Declarative Tool  (<form toolname="checkout_order" toolautosubmit>)
│   └── checkout_order      browser generates schema from form markup
│
├── Agent Lifecycle Events  (window.addEventListener)
│   ├── toolactivated       opens checkout modal · shows agent banner
│   └── toolcancel          closes modal · clears banner
│
└── SubmitEvent Handler  (#checkoutForm submit)
    ├── e.agentInvoked      branches between agent path and human path
    └── e.respondWith()     returns structured order result to the agent
```

---

## Best Practices

### Naming and descriptions

Use verb-first names that describe exactly what happens: `search_products`, `add_to_cart`, `checkout_order`. Write descriptions in positive terms — say what the tool does, not what it avoids. Specify when to use the tool:

```
// ✅ Good
"Search the product catalog by keyword. Use this when the user asks to find, 
browse, or filter products."

// ❌ Avoid
"Do not use this for adding items. Do not use for checkout."
```

### Schema design

Accept raw user input — do not ask the agent to perform calculations. If the user says "two iPhones", the `quantity` parameter should accept `2` directly, not a computed value. Use `enum` for fields with a fixed set of values. Enumerate valid values in the parameter `description` so errors are self-explanatory.

### Error handling

Validate inputs in code, not just in the schema — schema constraints are hints, not guarantees. Return descriptive errors that include the valid options so the model can self-correct without involving the user:

```javascript
if (!products.find(p => p.id === productId)) {
  return {
    content: [{
      type: "text",
      text: `Product "${productId}" not found. Valid IDs: iphone-15, galaxy-s24, pixel-9, macbook-air.`
    }]
  };
}
```

### `registerTool` vs `provideContext`

Always use `registerTool` when declarative tools are also present on the page. `provideContext` replaces the entire registered tool set — including tools the browser registered from your HTML forms. Mixing the two will silently delete declarative tools.

### UI synchronisation

After a tool's `execute` function returns, the agent may inspect the page state to verify the action succeeded. Always update the UI before returning from `execute`. If your update is asynchronous, await it inside the execute function before returning.

