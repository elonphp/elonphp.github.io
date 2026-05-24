---
title: "OCAdmin (English)"
description: "Design notes, implementation insights, and decision records for OCAdmin — a Laravel-based admin system inspired by OpenCart's backend."
build:
  list: never
---

> [→ 繁體中文版](/ocadmin/)

**OCAdmin** is a backend management system I built on Laravel, with its UI structure borrowed from OpenCart's admin panel.

This section contains **plain-language project notes**:

- How the system is designed and *why* it's designed that way
- Pitfalls encountered and trade-offs made during implementation
- Notes on integrating with the existing OpenCart ecosystem
- Design rationale behind each module

For technical details (APIs, configuration, CLI commands), please refer to the `docs/` folder in the GitHub repo. The content here focuses on **"why"** rather than **"how"**.

---

## Series articles

1. [Introduction: Why I Chose OpenCart Admin as the Design Blueprint](/ocadmin/en/introduction/)
2. [Overall Architecture: How We Split Portal, Core, and Module — and Why](/ocadmin/en/architecture/)
3. [Multilingual Mechanism: Three Independent Layers — UI, URL, and Content](/ocadmin/en/multilingual/)
4. [Naming Conventions: Singular/Plural (Horizontal) and Cross-Layer Alignment (Vertical)](/ocadmin/en/naming/)
5. [Permission Mechanism: Four-Segment Naming, Role Design, and Prefix Decoupling](/ocadmin/en/permissions/)
6. [Settings Mechanism: What You Need to Solve Once Settings Live in the DB](/ocadmin/en/settings/)
7. [Menu Mechanism: Permission Filtering, Structural Independence, and Two Approaches (Code / DB)](/ocadmin/en/menu/)

---

Each post is designed to stand alone — pick whichever interests you, or start with #1 if this is your first visit.
