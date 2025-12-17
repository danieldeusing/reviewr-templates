# Reviewr Templates

Official template repository for [Reviewr](https://github.com/danieldeusing/reviewr) - AI-assisted code review desktop application.

## Overview

This repository contains community and official templates including:

- **Skills** - Framework-specific code review guidelines
- **Hooks** - Pre/post commit hooks for automated checks
- **Agents** - Specialized AI agents for specific review tasks
- **Presets** - Curated combinations of skills, hooks, and agents

## Structure

```
reviewr-templates/
├── manifest.json              # Index of all templates
├── skills/                    # Code review skills by framework
│   ├── angular-19/
│   │   ├── meta.json         # Template metadata
│   │   └── skill.md          # Review guidelines
│   ├── react-18/
│   └── django-5/
├── hooks/                     # Automation hooks
│   ├── security-scan/
│   └── lint-check/
├── agents/                    # Specialized review agents
│   └── security-reviewer/
└── presets/                   # Combined configurations
    └── fullstack-angular/
```

## Using Templates

### In Reviewr

Templates can be browsed and installed directly from the Reviewr application:

1. Open Reviewr
2. Go to Settings → Templates
3. Browse available templates
4. Click "Install" on desired templates

### Manual Installation

Clone this repository and copy the desired templates to your Reviewr configuration:

```bash
# Clone the repository
git clone https://github.com/danieldeusing/reviewr-templates.git

# Copy a skill to your project
cp -r reviewr-templates/skills/angular-19 ~/.reviewr/skills/
```

## Template Types

### Skills

Skills provide framework-specific code review guidelines. Each skill includes:

- `meta.json` - Metadata and detection patterns
- `skill.md` - Detailed review guidelines and checklists

**Available Skills:**
- `angular-19` - Angular 19 with signals and standalone components
- `react-18` - React 18 with hooks and concurrent features (coming soon)
- `django-5` - Django 5 with async views (coming soon)

### Hooks

Hooks run automated checks at specific points in the review workflow.

**Available Hooks:**
- `security-scan` - Security vulnerability scanning (coming soon)
- `lint-check` - Linting validation (coming soon)

### Agents

Specialized AI agents for specific review tasks.

**Available Agents:**
- `security-reviewer` - Security-focused code reviews (coming soon)

### Presets

Presets combine multiple templates for common project types.

**Available Presets:**
- `fullstack-angular` - Angular + Node.js full-stack (coming soon)

## Creating Templates

### Skill Template

1. Create a directory under `skills/`
2. Add `meta.json` with template metadata
3. Add `skill.md` with review guidelines

```json
// meta.json
{
  "name": "my-framework",
  "displayName": "My Framework",
  "version": "1.0.0",
  "description": "Code review skill for My Framework",
  "category": "frontend",
  "language": "TypeScript",
  "framework": "MyFramework",
  "frameworkVersion": "1.x",
  "tags": ["myframework", "typescript"],
  "author": "Your Name",
  "license": "MIT",
  "detectionPatterns": {
    "files": ["myframework.config.js"],
    "dependencies": ["@myframework/core"]
  }
}
```

### Hook Template

1. Create a directory under `hooks/`
2. Add `meta.json` with hook metadata
3. Add `hook.js` or `hook.sh` with hook implementation

### Contributing

We welcome contributions! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## License

MIT License - see [LICENSE](LICENSE) for details.
