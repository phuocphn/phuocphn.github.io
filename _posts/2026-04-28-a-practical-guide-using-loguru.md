---
layout: post
title: A Practical Guide to Using Loguru Across Multiple Python Modules
---

{{ page.title }}
================

<p class="meta">28 April 2026 - Munich</p>

Logging across multiple modules in Python can get messy quickly—especially with the standard `logging` module. Fortunately, **Loguru** simplifies things with a clean, global logger and minimal boilerplate.

This post walks through:
- A clean multi-module setup
- Per-module log levels
- Logging each module into separate files
- Subtle Python pitfalls (and how to avoid them)

---


## 1. Basic Multi-Module Setup
Create a central logger configuration and initialize it **once**.

```python
# logger.py
from loguru import logger
import sys

def setup_logger():
    logger.remove()
    logger.add(sys.stdout, level="INFO",
               format="<green>{time:YYYY-MM-DD HH:mm:ss}</green> | <level>{level: <8}</level> | <cyan>{name}</cyan>:<cyan>{function}</cyan>:<cyan>{line}</cyan> - <level>{message}</level>")
    logger.add("logs/app.log", rotation="10 MB", retention="10 days", compression="zip",
               level="DEBUG",
               format="{time:YYYY-MM-DD HH:mm:ss} | {level} | {name}:{function}:{line} - {message}")
    return logger

# main.py
from logger import setup_logger
logger = setup_logger()

# module_a.py / module_b.py
from loguru import logger
logger.info("Hello from module")
```

**Key idea:** Loguru uses a global singleton logger → configure once, use everywhere.
**Important**: Do NOT call setup_logger() multiple times in different modules. Just import logger directly

---

## 2. Per-Module Log Levels

Loguru doesn’t support per-module levels out of the box, but filters solve it.

```python
from loguru import logger
import sys

MODULE_LEVELS = {"module_a": "DEBUG", "module_b": "INFO"}

def module_filter(record):
    name = record["name"]
    level = record["level"].name
    allowed = MODULE_LEVELS.get(name, "INFO")
    return logger.level(level).no >= logger.level(allowed).no

logger.remove()
logger.add(sys.stdout, level="DEBUG", filter=module_filter)
```

---

## 3. Logging Each Module to Its Own File

You can route logs into separate files using one handler per module.

```python
from loguru import logger
import sys
from pathlib import Path

LOG_DIR = Path("logs"); LOG_DIR.mkdir(exist_ok=True)
MODULE_LEVELS = {"module_a": "DEBUG", "module_b": "INFO"}

logger.remove()
logger.add(sys.stdout, level="INFO",
           format="{time:HH:mm:ss} | {level} | {name} - {message}")

for module, level in MODULE_LEVELS.items():
    logger.add(LOG_DIR / f"{module}.log",
               level=level,
               rotation="5 MB",
               retention="7 days",
               compression="zip",
               filter=lambda record, m=module: record["name"].endswith(m),
               format="{time:YYYY-MM-DD HH:mm:ss} | {level} | {name}:{function}:{line} - {message}")

logger.add(LOG_DIR / "app.log",
           level="INFO",
           rotation="10 MB",
           retention="10 days",
           compression="zip")
```

**Result:**
```
logs/
├── module_a.log
├── module_b.log
└── app.log
```

---

## 4. Understanding the Filter Lambda

```python
filter=lambda record, m=module: record["name"].endswith(m)
```

### What it means:
- `record["name"]` → module name (e.g., `"module_a"` or `"myproj.module_a"`)
- `.endswith(m)` → matches module reliably
- `m=module` → **captures the loop variable correctly**

### Why `m=module` matters:

Without it:
```python
lambda record: record["name"].endswith(module)
```

All filters would use the **last value** of `module` (Python late binding issue).

---

## 5. A More Robust Alternative (Recommended)

Avoid relying on module names entirely:

```python
# module_a.py
from loguru import logger
logger = logger.bind(module="A")

# logger setup
filter=lambda record, m=module: record["extra"].get("module") == m
```

This avoids import-path inconsistencies like:
- `"module_a"`
- `"myproject.module_a"`

---

## 6. Common Pitfalls

- Calling `logger.add()` in multiple modules → duplicate logs  
- Forgetting `logger.remove()` → duplicated output  
- Matching module names with `==` instead of `.endswith()`  
- Ignoring Python’s lambda scoping behavior  

---

## 7. Takeaways

- Loguru simplifies multi-module logging with a global logger  
- Use **filters** for per-module control  
- Use **one handler per module** for separate log files  
- Always handle lambda scoping (`m=module`) correctly  
- Consider `logger.bind()` for larger systems  

---

## Final Thought

This setup scales well from small scripts to research codebases (e.g., ML experiments, RL pipelines), while staying readable and maintainable—exactly where Loguru shines.
```