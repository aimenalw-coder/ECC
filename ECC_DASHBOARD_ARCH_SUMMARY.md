# ECC Dashboard — Architecture Summary

> **Status:** Manual extraction. GitNexus cannot parse `ecc_dashboard.py` due to a
> tree-sitter "Invalid argument" error (reproduced on both incremental and `--force`
> re-index runs). This document is the authoritative source of truth for the structure
> of this file until the parser issue is resolved upstream.
>
> Last extracted: 2026-06-10 from `ecc_dashboard.py` (932 lines) and
> `scripts/lib/ecc_dashboard_runtime.py` (80 lines).

---

## Files

| File | Lines | Role |
|------|-------|------|
| `ecc_dashboard.py` | 932 | Main application — data loaders + `ECCDashboard` god-class + `main()` |
| `scripts/lib/ecc_dashboard_runtime.py` | 80 | Platform-specific helpers (terminal launch, window maximization) |

---

## Dependencies

### Standard Library
| Import | Used for |
|--------|----------|
| `tkinter` (`tk`, `ttk`, `scrolledtext`, `messagebox`) | All UI widgets and dialogs |
| `tkinter.filedialog` | Directory picker in `browse_path()` (imported inline) |
| `os` | Path operations throughout |
| `json` | Imported but unused in current code |
| `pathlib.Path` | `Path(target).as_uri()` in `_open_project_doc()` |
| `typing.Dict`, `List`, `Optional` | Type hints on all loader functions |
| `logging` | Debug-level logging in loaders and `apply_theme()` |
| `webbrowser` | Opens project docs in `_open_project_doc()` |

### Internal
| Import | Symbols | Used in |
|--------|---------|---------|
| `scripts.lib.ecc_dashboard_runtime` | `launch_terminal`, `maximize_window` | `open_terminal()`, `__init__()` |

---

## Module-Level Functions (Data Loaders)

All loaders follow the same pattern: scan a project subdirectory, parse files, return
a `List[Dict]`. Each has a hardcoded fallback list returned when the directory is absent
(these fallbacks are likely stale and should not be relied upon).

| Function | Line | Scans | Returns |
|----------|------|-------|---------|
| `get_project_path()` | L24 | — | `str` — `dirname(abspath(__file__))` |
| `load_agents(project_path)` | L29 | `agents/*.md` | `List[Dict]` — keys: `name`, `purpose`, `when_to_use`, `path` |
| `load_skills(project_path)` | L93 | `skills/*/SKILL.md` | `List[Dict]` — keys: `name`, `description`, `category`, `path` |
| `load_commands(project_path)` | L173 | `commands/*.md` | `List[Dict]` — keys: `name`, `description` |
| `load_rules(project_path)` | L222 | `rules/common/*.md`, `rules/<lang>/*.md` | `List[Dict]` — keys: `name`, `language`, `path` |

**`load_agents` detail:** Parses YAML frontmatter (`name:`, `description:`) between
`---` delimiters. Falls back on hardcoded 19-agent list if `agents/` directory not found.

**`load_skills` detail:** Reads first non-blank line of `SKILL.md` as description.
Derives `category` from name substring matching (e.g. `python` → Python, `react` → Frontend).

---

## Class: `ECCDashboard(tk.Tk)`

**File:** `ecc_dashboard.py:272`
**Inherits:** `tk.Tk` — the class IS the application window, not just a controller.

### Instance Attributes (set in `__init__`)

| Attribute | Type | Source |
|-----------|------|--------|
| `self.project_path` | `str` | `get_project_path()` |
| `self.agents` | `List[Dict]` | `load_agents()` |
| `self.skills` | `List[Dict]` | `load_skills()` |
| `self.commands` | `List[Dict]` | `load_commands()` |
| `self.rules` | `List[Dict]` | `load_rules()` |
| `self.settings` | `Dict` | Inline literal — `{'project_path': ..., 'theme': 'light'}` |
| `self.notebook` | `ttk.Notebook` | `create_widgets()` |
| `self.agent_tree` | `ttk.Treeview` | `create_agents_tab()` |
| `self.agent_search` | `ttk.Entry` | `create_agents_tab()` |
| `self.agent_count_label` | `ttk.Label` | `create_agents_tab()` |
| `self.agent_details` | `scrolledtext.ScrolledText` | `create_agents_tab()` |
| `self.skill_tree` | `ttk.Treeview` | `create_skills_tab()` |
| `self.skill_search` | `ttk.Entry` | `create_skills_tab()` |
| `self.skill_category` | `ttk.Combobox` | `create_skills_tab()` |
| `self.skill_count_label` | `ttk.Label` | `create_skills_tab()` |
| `self.skill_details` | `scrolledtext.ScrolledText` | `create_skills_tab()` |
| `self.command_tree` | `ttk.Treeview` | `create_commands_tab()` |
| `self.rules_tree` | `ttk.Treeview` | `create_rules_tab()` |
| `self.rules_language` | `ttk.Combobox` | `create_rules_tab()` |
| `self.path_entry` | `ttk.Entry` | `create_settings_tab()` |
| `self.theme_var` | `tk.StringVar` | `create_settings_tab()` |
| `self.font_var` | `tk.StringVar` | `create_settings_tab()` |
| `self.size_var` | `tk.StringVar` | `create_settings_tab()` |
| `self.status_label` | `ttk.Label` | `create_widgets()` |
| `self.title_label` | `ttk.Label` | `create_widgets()` |
| `self.version_label` | `ttk.Label` | `create_widgets()` |

### Methods

#### Initialization / Window

| Method | Line | Calls | Notes |
|--------|------|-------|-------|
| `__init__(self)` | L275 | `get_project_path`, all 4 loaders, `setup_styles`, `create_widgets`, `maximize_window` | Loads data eagerly; calls `maximize_window` (external), not `center_window` |
| `setup_styles(self)` | L309 | `ttk.Style` | Configures `clam` theme; sets fonts for Notebook, Treeview, Button |
| `center_window(self)` | L326 | `self.update_idletasks`, `self.geometry` | **Dead method** — never called; superseded by `maximize_window()` |
| `create_widgets(self)` | L335 | `create_agents_tab`, `create_skills_tab`, `create_commands_tab`, `create_rules_tab`, `create_settings_tab` | Builds main frame, header (logo + title + version), notebook, status bar |

#### Agents Tab

| Method | Line | Calls | Notes |
|--------|------|-------|-------|
| `create_agents_tab(self)` | L381 | `populate_agents`, binds `filter_agents`, `on_agent_select` | Search bar + treeview (name, purpose) + detail panel |
| `populate_agents(self, agents)` | L438 | `agent_tree.insert` | Clears then repopulates treeview |
| `filter_agents(self, event=None)` | L446 | `populate_agents` | Filters `self.agents` by name/purpose substring |
| `on_agent_select(self, event)` | L459 | `agent_details.insert` | Shows name, purpose, when_to_use, usage hint in text area |

#### Skills Tab

| Method | Line | Calls | Notes |
|--------|------|-------|-------|
| `create_skills_tab(self)` | L486 | `get_categories`, `populate_skills`, binds `filter_skills`, `on_skill_select` | Search + category combobox + treeview (name, category, description) + details |
| `get_categories(self)` | L549 | — | Derives sorted unique category list from `self.skills` |
| `populate_skills(self, skills)` | L554 | `skill_tree.insert` | Clears then repopulates treeview |
| `filter_skills(self, event=None)` | L563 | `populate_skills` | Filters by category then search substring |
| `on_skill_select(self, event)` | L580 | `skill_details.insert` | Shows name, category, description, path, usage hint |

#### Commands Tab

| Method | Line | Calls | Notes |
|--------|------|-------|-------|
| `create_commands_tab(self)` | L608 | `command_tree.insert` | Read-only display — no selection handler, no filter, no detail panel |

#### Rules Tab

| Method | Line | Calls | Notes |
|--------|------|-------|-------|
| `create_rules_tab(self)` | L651 | `get_rule_languages`, `populate_rules`, binds `filter_rules` | Language filter combobox + treeview (name, language) |
| `get_rule_languages(self)` | L699 | — | Derives sorted unique language list from `self.rules` |
| `populate_rules(self, rules)` | L704 | `rules_tree.insert` | Clears then repopulates treeview |
| `filter_rules(self, event=None)` | L713 | `populate_rules` | Filters `self.rules` by language |

#### Settings Tab

| Method | Line | Calls | Notes |
|--------|------|-------|-------|
| `create_settings_tab(self)` | L728 | `browse_path`, `open_terminal`, `open_readme`, `open_agents`, `refresh_data`, `apply_theme` | Project path, theme radios, font/size combos, quick actions, about text |
| `browse_path(self)` | L802 | `filedialog.askdirectory` | Updates `path_entry` |
| `open_terminal(self)` | L810 | `launch_terminal` | Reads `path_entry`, delegates to runtime helper |
| `_open_project_doc(self, filename)` | L818 | `webbrowser.open`, `messagebox.showerror` | **Path-traversal guard** via `commonpath` check before opening |
| `open_readme(self)` | L830 | `_open_project_doc` | Opens `README.md` |
| `open_agents(self)` | L834 | `_open_project_doc` | Opens `AGENTS.md` |
| `refresh_data(self)` | L838 | all 4 loaders, `populate_agents`, `populate_skills` | **Bug:** does not call `populate_commands` or `populate_rules` — those tabs do not refresh |
| `apply_theme(self)` | L863 | `update_widget_colors` (inner fn), `ttk.Style` | Recursive widget walk to apply bg/fg colors; applies font to `title_label` and `version_label` directly |

---

## Entry Point

```python
def main():          # L924
    app = ECCDashboard()
    app.mainloop()

if __name__ == "__main__":
    main()           # L930
```

`main()` is the sole external caller of `ECCDashboard`. No other file imports or
instantiates it.

---

## `scripts/lib/ecc_dashboard_runtime.py` — Full API

| Function | Line | Signature | Notes |
|----------|------|-----------|-------|
| `maximize_window(window)` | L14 | `(window) -> None` | macOS: `geometry(WxH+0+0)`; Windows: `state('zoomed')`; Linux: `attributes('-zoomed', True)` |
| `build_terminal_launch(path, *, os_name, system_name)` | L44 | `(str, *, Optional[str], Optional[str]) -> Tuple[List[str], Dict]` | Returns safe `argv`/`kwargs` for `subprocess.Popen`; no `shell=True` |
| `launch_terminal(path)` | L73 | `(str) -> None` | Validates `path` is real directory via `os.path.realpath`; calls `build_terminal_launch` then `Popen` |

---

## Known Issues (found during extraction)

| # | Severity | Location | Description |
|---|----------|----------|-------------|
| 1 | Medium | `refresh_data()` L838 | Commands and Rules tabs are **not repopulated** on refresh — only Agents and Skills call their `populate_*` methods. Tab labels update but data is stale. |
| 2 | Low | `center_window()` L326 | Dead method — defined but never called. `maximize_window()` in `__init__` supersedes it entirely. |
| 3 | Low | `json` import L10 | Imported but never used. |
| 4 | Low | All 4 loaders | Hardcoded fallback lists are likely stale and will silently serve incorrect data if a directory is missing. |
| 5 | Low | `apply_theme()` L863 | Recursive widget walk is fragile and not the canonical ttk theming approach; widgets added after `apply_theme()` runs will not receive the current theme. |

---

## Refactoring Target Map

See `CLAUDE.md → Architectural Priorities` for the full decomposition mandate.
Quick reference:

| Current location | Extract to |
|-----------------|------------|
| `load_agents`, `load_skills`, `load_commands`, `load_rules`, `get_project_path` | `dashboard/loaders.py` |
| `create_agents_tab`, `populate_agents`, `filter_agents`, `on_agent_select` | `dashboard/agents_tab.py` (`AgentsTab`) |
| `create_skills_tab`, `get_categories`, `populate_skills`, `filter_skills`, `on_skill_select` | `dashboard/skills_tab.py` (`SkillsTab`) |
| `create_commands_tab` | `dashboard/commands_tab.py` (`CommandsTab`) |
| `create_rules_tab`, `get_rule_languages`, `populate_rules`, `filter_rules` | `dashboard/rules_tab.py` (`RulesTab`) |
| `create_settings_tab`, `browse_path`, `open_terminal`, `open_readme`, `open_agents`, `apply_theme` | `dashboard/settings_tab.py` (`SettingsTab`) |
| `ECCDashboard` (slim: `__init__`, `setup_styles`, `center_window`, `create_widgets`, `refresh_data`) | `dashboard/app.py` |
| `main()` + `if __name__` block | `ecc_dashboard.py` (entry point only) |
