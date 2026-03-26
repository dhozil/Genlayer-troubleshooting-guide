#🔧 GenLayer Testnet Bradbury — Developer Troubleshooting Guide

*A practical guide based on real errors encountered while building Intelligent Contracts on Testnet Bradbury.*

---

## Introduction

This guide covers the most common errors you'll encounter when deploying Intelligent Contracts on GenLayer Testnet Bradbury. Every error in this guide was encountered and solved during real development — not theoretical scenarios.

If you're coming from Testnet Asimov, pay close attention. Bradbury uses a different SDK version and many syntax patterns have changed.

---

## 1. "Could Not Load Contract Schema"

When it happens: Before deployment, when Studio tries to parse your contract.

Root causes and fixes:

### ❌ Wrong Depends hash
The most common cause. Asimov and Bradbury use different SDK hashes.
# ❌ Wrong — Asimov hash
# { "Depends": "py-genlayer:test" }

# ✅ Correct — Bradbury hash
# { "Depends": "py-genlayer:15qfivjvy80800rh998pcxmd2m8va1wq2qzqhz850n8ggcr4i9q0" }

### ❌ Unsupported return type dict
GenLayer schema parser does not support dict as a return type or state variable.
# ❌ Wrong
@gl.public.view
def get_result(self) -> dict:
    return {"status": self.status}

# ✅ Correct — split into separate view methods
@gl.public.view
def get_status(self) -> str:
    return self.status

### ❌ Unsupported state variable type Address
# ❌ Wrong
owner: Address

def __init__(self):
    self.owner = gl.message.sender_address  # AttributeError: 'Address' has no attribute 'encode'

# ✅ Correct — remove owner or convert to str manually
# Simply don't store sender_address if not needed

### ❌ Using bool return type inconsistently
Stick to str return types when in doubt. bool can cause schema issues in some contexts.
# ✅ Safe pattern
@gl.public.view
def is_verified(self) -> str:
    return "true" if self.status == "verified" else "false"

---

## 2. VM_ERROR: Module Attribute Errors

These errors happen at runtime after successful deployment.

### ❌ module 'genlayer.gl' has no attribute 'dataclass'
# ❌ Wrong — gl.dataclass does not exist
@gl.dataclass
class MyData:
    name: str

# ✅ Correct — use plain state variables directly in the contract class
class MyContract(gl.Contract):
    name: str
    status: str

### ❌ module 'genlayer.std' has no attribute 'eq_principle'
# ❌ Wrong — missing 's' at the end
result = gl.eq_principle.prompt_non_comparative(...)

# ✅ Correct — note the 's' at the end
result = gl.eq_principles.eq_principle_prompt_non_comparative(...)

### ❌ module 'genlayer.std.eq_principles' has no attribute 'prompt_non_comparative'
# ❌ Wrong — method name changed in Bradbury
result = gl.eq_principles.prompt_non_comparative(...)

# ✅ Correct — full method name required
result = gl.eq_principles.eq_principle_prompt_non_comparative(...)

### ❌ module 'genlayer.std' has no attribute 'nondet'
# ❌ Wrong — nondet module doesn't exist in this SDK version
page = gl.nondet.web.render(url, mode="text")
result = gl.nondet.exec_prompt(prompt)

# ✅ Correct
page = gl.get_webpage(url, mode="text")

---

## 3. Correct Contract Template for Bradbury

Use this as your starting point for every new contract:
# v0.1.0
# { "Depends": "py-genlayer:15qfivjvy80800rh998pcxmd2m8va1wq2qzqhz850n8ggcr4i9q0" }
from genlayer import *
import json

class MyContract(gl.Contract):
    state: str  # Always use str for state variables when possible

    def __init__(self):  # No parameters in __init__
        self.state = "initial"

    @gl.public.write
    def my_write_method(self, input: str) -> None:
        def non_det():
            # All external calls go INSIDE this function
            return gl.get_webpage("https://example.com", mode="text")

        result = gl.eq_principles.eq_principle_prompt_non_comparative(
            non_det,
            task="Your task description here. Input: " + input,
            criteria="Your criteria for validators to reach consensus."
        )
        self.state = str(result).strip()

    @gl.public.view
[26/03/2026 09:42] DhoziL: def get_state(self) -> str:
        return self.state

---

## 4. Correct eq_principles Usage

The eq_principles module is the core of GenLayer's AI consensus. Here's the correct pattern:
def my_write_method(self, query: str) -> None:
    # Step 1: Define non-deterministic function
    def non_det():
        # Fetch external data here
        return gl.get_webpage("https://api.example.com?q=" + query, mode="text")

    # Step 2: Call eq_principle with task + criteria
    result = gl.eq_principles.eq_principle_prompt_non_comparative(
        non_det,           # ← your non-det function
        task="...",        # ← what validators should do with the data
        criteria="..."     # ← how validators reach consensus
    )

    # Step 3: Store result as string
    self.state = str(result).strip()

Rules:
- non_det function must be defined inside the write method
- All gl.get_webpage() calls must be inside non_det
- task and criteria must be plain strings — no f-strings with variables
  - Instead concatenate: "Check if " + variable + " is valid"
- Always call str(result).strip() on the output

---

## 5. Validator Disagreement Errors

When it happens: Validators reach different conclusions and cannot agree.

Why it happens:
- External API returns slightly different data to different validators
- LLM responses vary between validators
- Network timeout during fetch

Solutions:

✅ Retry — simply call the method again. Usually resolves on 2nd or 3rd attempt.

✅ Simplify your task — ask for binary answers when possible:# ❌ Complex task — harder to reach consensus
task="Extract all details about the price including high, low, and volume"

# ✅ Simple task — easier consensus
task="What is the current price of bitcoin? Respond with number only."
criteria="Answer must be a number only."

✅ Use stricter criteria:
criteria="Answer must be exactly: true or false"
# Better than:
criteria="Answer should indicate yes or no"

---

## 6. Web Fetch Issues

### JavaScript-rendered pages
Some pages (like Credly badge pages) require JavaScript to load content. GenLayer fetches static HTML only.
# ❌ Will return empty or incomplete content
gl.get_webpage("https://www.credly.com/badges/some-badge", mode="text")

# ✅ Use API endpoints or static pages instead
gl.get_webpage("https://api.coingecko.com/api/v3/simple/price?ids=bitcoin&vs_currencies=usd", mode="text")
gl.get_webpage("https://wttr.in/Jakarta?format=j1", mode="text")

### Tips for reliable fetches:
- Prefer JSON API endpoints over HTML pages
- Use mode="text" always
- Keep URLs short and direct
- Avoid URLs that require authentication or cookies

---

## 7. Quick Reference: Bradbury vs Asimov

| Feature | Asimov | Bradbury |
|---|---|---|
| Depends hash | 1jb45aa8... or test | 15qfivjvy8... |
| Web fetch | gl.nondet.web.render() | gl.get_webpage() |
| AI prompt | gl.nondet.exec_prompt() | Inside eq_principles |
| Consensus | gl.eq_principle.strict_eq() | gl.eq_principles.eq_principle_prompt_non_comparative() |
| Init method | def init(self) | def __init__(self) |
| State dict | Supported | ❌ Not supported |
| Return dict | Supported | ❌ Not supported |

---

## 8. Minimal Working Example

Copy this, deploy it, and confirm it works before building anything complex:
# v0.1.0
# { "Depends": "py-genlayer:15qfivjvy80800rh998pcxmd2m8va1wq2qzqhz850n8ggcr4i9q0" }
from genlayer import *
import json

class HelloBradbury(gl.Contract):
    state: str

    def __init__(self):
        self.state = "ready"

    @gl.public.write
    def fetch_btc_price(self) -> None:
        def non_det():
            return gl.get_webpage(
                "https://api.coingecko.com/api/v3/simple/price?ids=bitcoin&vs_currencies=usd",
                mode="text"
            )
        result = gl.eq_principles.eq_principle_prompt_non_comparative(
            non_det,
            task="Extract the bitcoin price in USD. Respond with the number only.",
            criteria="Answer must be a number only."
        )
        self.state = str(result).strip()

    @gl.public.view
    def get_state(self) -> str:
    return self.state

If this works, your environment is set up correctly and you can start building.

---

*Written based on real development experience on GenLayer Testnet Bradbury.*
*Part of the GenLayer Incentivized Builder Program contributions.*
