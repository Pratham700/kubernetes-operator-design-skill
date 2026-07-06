# Kubernetes Operator Design Skill

A high-profile, production-grade custom skill designed for AI Coding Agents (such as Google Antigravity, Claude Code, and Codex / GitHub Copilot) to guide the architecture, development, security hardening, and distribution of Kubernetes Operators.

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

## Installation & Setup

### 1. Google Antigravity Installation
Antigravity automatically discovers skills placed in its customization roots.

#### Global Scope (Available in all your projects)
1. Clone this repository locally:
   ```bash
   git clone https://github.com/Pratham700/kubernetes-operator-design-skill.git ~/kubernetes-operator-design-skill
   ```
2. Symlink the skill folder to your global config directory:
   ```bash
   ln -s ~/kubernetes-operator-design-skill/skills/kubernetes-operator-design ~/.gemini/config/skills/kubernetes-operator-design
   ```

#### Workspace Scope (Specific to your active project)
Copy the skill folder into your repository's workspace customizations root:
```bash
mkdir -p .agents/skills/
cp -r ~/kubernetes-operator-design-skill/skills/kubernetes-operator-design .agents/skills/
```

---

### 2. Claude Code Installation
Claude Code uses `CLAUDE.md` files to enforce persistent instructions and conventions across sessions.

#### Global Scope (Enforced in all Claude Code sessions)
Append the content of `SKILL.md` (and references if needed) to Claude's global configurations file:
```bash
mkdir -p ~/.claude/
cat ~/kubernetes-operator-design-skill/skills/kubernetes-operator-design/SKILL.md >> ~/.claude/CLAUDE.md
```

#### Project Scope (Committed to your Git repository)
Create a project-specific instructions file in your workspace root:
```bash
cat ~/kubernetes-operator-design-skill/skills/kubernetes-operator-design/SKILL.md > ./CLAUDE.md
```

---

### 3. Codex & GitHub Copilot Installation
GitHub Copilot reads instructions from `.github/copilot-instructions.md` to customize its context and ensure code output matches project guidelines.

#### Project Scope (Applies to all Copilot Chat requests in the IDE)
Create the custom instructions file in your repository's `.github` directory:
1. Create the target directory:
   ```bash
   mkdir -p .github/
   ```
2. Copy the skill workflow as the baseline instructions:
   ```bash
   cat ~/kubernetes-operator-design-skill/skills/kubernetes-operator-design/SKILL.md > .github/copilot-instructions.md
   ```
3. (Optional) To add all reference sub-files to Copilot's context, merge them into a single consolidated file or place them under `.github/instructions/` and configure glob mappings.

---

## License

Distributed under the Apache 2.0 License. See `LICENSE` for more information.
