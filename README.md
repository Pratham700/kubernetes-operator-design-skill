# Kubernetes Operator Design Skill

A high-profile, production-grade custom skill designed for AI Coding Agents (such as Google Antigravity, Claude Code, and Codex) to guide the architecture, development, security hardening, and distribution of Kubernetes Operators.

This skill synthesizes core principles from O'Reilly's *Kubernetes Operators* with advanced security guidelines for informer cache tuning and OOMKill protection.

---

## Features & Content Scopes

1. **SRE Philosophy & The Seven Habits**: Encodes the CoreOS/Red Hat standard for building operators as automated Site Reliability Engineers (decoupled lifecycles, leveraging native abstractions, chaos testing).
2. **Reconciliation & Finalizers**: Clean, production-ready Go templates using the `controller-runtime` library for idempotent loops, parent-child owner bindings, and finalizer-driven deletions.
3. **Memory Exhaustion (OOMKill) Prevention**: Critical configurations to prevent DoS caching attacks (label-filtering informers, metadata-only watches, stripping managed fields, avoiding typed/unstructured cache traps, and using uncached API readers).
4. **Least-Privilege RBAC & Security Contexts**: Guidance on converting wildcards to scoped Roles/RoleBindings, namespace scoping, and OS-level security constraints.
5. **OLM Metadata Specification**: Rules for writing the ClusterServiceVersion (CSV), descriptors (`specDescriptors`, `statusDescriptors`) for UI bindings, install modes, and release upgrades channels.
6. **Maturity Model & Golden Signals**: Framework for progressing an operator from simple installation to automated auto-pilot, with telemetry for SRE's Four Golden Signals (Latency, Traffic, Errors, Saturation).

---

## Repository Structure

```text
.
├── LICENSE                 # Apache 2.0 Open Source License
├── README.md               # Visual instructions and installation guide
└── skills/
    └── kubernetes-operator-design/
        ├── SKILL.md        # The main skill definition (under 500 lines)
        └── references/     # Supplemental design specifications
            ├── seven-habits.md
            ├── reconciliation.md
            ├── maturity-model.md
            ├── cache-optimization.md
            ├── rbac-security.md
            └── olm-packaging.md
```

---

## Installation

### 1. Global Scope (Available in all your projects)
To make this skill globally available to your AI assistant:
1. Clone this repository to a local folder:
   ```bash
   git clone https://github.com/Pratham700/kubernetes-operator-design-skill.git ~/kubernetes-operator-design-skill
   ```
2. Create a symbolic link from the skill directory to your global agent configurations:
   ```bash
   ln -s ~/kubernetes-operator-design-skill/skills/kubernetes-operator-design ~/.gemini/config/skills/kubernetes-operator-design
   ```

### 2. Workspace Scope (Shared with your project team)
To commit this skill directly to a specific codebase workspace:
1. Copy or clone the skill folder into your repository's `.agents` configurations:
   ```bash
   mkdir -p .agents/skills/
   cp -r ~/kubernetes-operator-design-skill/skills/kubernetes-operator-design .agents/skills/
   ```
2. Add `.agents` to git so your team can use the same skill workflows.

---

## License

Distributed under the Apache 2.0 License. See `LICENSE` for more information.
