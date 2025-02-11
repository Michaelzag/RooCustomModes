# Custom Modes and Rules Implementation Guide

## Overview

This document provides a detailed explanation of how custom modes and rules are implemented in the Roo-Code project, including examples and limitations.

## Custom Modes Implementation

### 1. Data Structure

Custom modes are defined using a Zod schema that enforces type safety and validation. Here's the core structure:

```typescript
const CustomModeSchema = z.object({
  slug: z.string().regex(/^[a-zA-Z0-9-]+$/),
  name: z.string().min(1),
  roleDefinition: z.string().min(1),
  customInstructions: z.string().optional(),
  groups: GroupsArraySchema,
})
```

#### Required Fields:
- `slug`: Unique identifier (letters, numbers, dashes only)
- `name`: Display name for the mode
- `roleDefinition`: Detailed description of the mode's role
- `groups`: Array of allowed tool groups

#### Optional Fields:
- `customInstructions`: Additional mode-specific instructions

### 2. Tool Groups

Tool groups define what actions a mode can perform. Groups can be specified in two ways:

```typescript
// Simple group specification
groups: ["read", "edit"]

// Group with file restrictions
groups: [
  ["edit", { 
    fileRegex: "\\.md$",
    description: "Markdown files only"
  }]
]
```

Available tool groups:
- `read`: File reading operations
- `edit`: File modification operations
- `browser`: Browser interactions
- `command`: CLI command execution
- `mcp`: MCP tool usage

### 3. Example Custom Mode

```json
{
  "customModes": [
    {
      "slug": "designer",
      "name": "Designer",
      "roleDefinition": "You are Roo, a UI/UX expert specializing in design systems and frontend development. Your expertise includes:\n- Creating and maintaining design systems\n- Implementing responsive and accessible web interfaces\n- Working with CSS, HTML, and modern frontend frameworks\n- Ensuring consistent user experiences across platforms",
      "groups": [
        "read",
        ["edit", { "fileRegex": "\\.(css|scss|html|jsx|tsx)$", "description": "Frontend files only" }],
        "browser"
      ],
      "customInstructions": "Always consider mobile-first design principles and accessibility guidelines."
    }
  ]
}
```

### 4. Management System

The CustomModesManager class provides several key operations:

```typescript
class CustomModesManager {
  async getCustomModes(): Promise<ModeConfig[]>
  async updateCustomMode(slug: string, config: ModeConfig): Promise<void>
  async deleteCustomMode(slug: string): Promise<void>
  async resetCustomModes(): Promise<void>
}
```

Key features:
- File-based storage in `cline_custom_modes.json`
- Write queue to prevent conflicts
- File watching for live updates
- VSCode globalState persistence
- Error handling and validation

## Rules Implementation

### 1. File Structure

The system supports both global and mode-specific rules through different file types:

```
project/
├── .clinerules                 # Global rules for all modes
├── .clinerules-code           # Code mode specific rules
├── .clinerules-architect      # Architect mode specific rules
├── .cursorrules               # Alternative rule format (supported)
└── .windsurfrules             # Alternative rule format (supported)
```

### 2. Rule File Examples

#### Global Rules (.clinerules)
```markdown
# Code Quality Rules

1. Test Coverage:
   - Before attempting completion, always make sure that any code changes have test coverage
   - Ensure all tests pass before submitting changes

2. Lint Rules:
   - Never disable any lint rules without explicit user approval
   - If a lint rule needs to be disabled, ask the user first and explain why
   - Prefer fixing the underlying issue over disabling the lint rule
```

#### Mode-Specific Rules (.clinerules-architect)
```markdown
# Architect Mode Rules

1. Design Principles:
   - Always document architectural decisions
   - Consider scalability in all design choices
   - Maintain separation of concerns

2. Review Requirements:
   - Validate requirements before implementation
   - Document assumptions and constraints
   - Consider security implications
```

### 3. Rule Loading Process

The rule loading system:
1. Checks for mode-specific rules first
2. Loads global rules
3. Combines them in the system prompt

```typescript
async function addCustomInstructions(
  modeCustomInstructions: string,
  globalCustomInstructions: string,
  cwd: string,
  mode: string,
  options: { preferredLanguage?: string } = {},
): Promise<string>
```

## Role Definition and Custom Instructions Technical Implementation

### System Prompt Integration

1. Role Definition (`roleDefinition`):
   - Appears at the very beginning of the system prompt
   - Sets the foundational behavior and capabilities
   - Implementation in system.ts:
     ```typescript
     const roleDefinition = promptComponent?.roleDefinition || modeConfig.roleDefinition
     const basePrompt = `${roleDefinition}\n\n${getSharedToolUseSection()}`
     ```

2. Custom Instructions (`customInstructions`):
   - Added at the end of the system prompt
   - Multiple sources combined in specific order:
     ```typescript
     const effectiveInstructions = [
       globalInstructions,
       modePrompt?.customInstructions
     ].filter(Boolean).join("\n\n")
     ```

### Technical Capabilities and Limitations

1. Role Definition:
   - Must be non-empty (Zod validation)
   - Can include:
     * Multi-line text with \n
     * Markdown formatting
     * Tool references
     * Capability descriptions
   - Cannot include:
     * Dynamic content
     * Runtime variables
     * System command execution
     * Direct tool group modifications

2. Custom Instructions:
   - Optional field
   - Can include:
     * Conditional behaviors
     * Tool usage patterns
     * Environment-specific instructions
     * Project-specific guidelines
   - Cannot:
     * Override core system behaviors
     * Modify tool permissions
     * Access system internals
     * Execute commands directly

### Runtime Behavior

1. Prompt Component Override System:
   ```typescript
   // In system.ts
   const modeConfig = getModeBySlug(mode, customModeConfigs) || 
                     modes.find((m) => m.slug === mode) || 
                     modes[0]
   const roleDefinition = promptComponent?.roleDefinition || 
                         modeConfig.roleDefinition
   ```
   This enables:
   - Dynamic role switching
   - Runtime personality adjustments
   - Task-specific modifications
   - Context-aware behavior

2. Instructions Layering:
   ```typescript
   // In custom-instructions.ts
   const sections = []
   if (options.preferredLanguage) {
     sections.push(`Language Preference:\nYou should always speak and think in the ${options.preferredLanguage} language.`)
   }
   if (globalCustomInstructions?.trim()) {
     sections.push(`Global Instructions:\n${globalCustomInstructions.trim()}`)
   }
   if (modeCustomInstructions?.trim()) {
     sections.push(`Mode-specific Instructions:\n${modeCustomInstructions.trim()}`)
   }
   ```

### Memory and Performance Impact

1. Context Window Considerations:
   - Role definitions and custom instructions consume tokens
   - Long definitions reduce available context for conversation
   - Multiple instruction layers can lead to truncation
   - System prompt size affects all interactions

2. Loading Performance:
   - Instructions are loaded on every mode switch
   - File reading operations for rules
   - Memory caching in VSCode globalState
   - Write queue for updates

### Advanced Usage Techniques

1. Dynamic Role Adaptation:
```json
{
  "roleDefinition": "You are Roo, dynamically adapting between:\n- Frontend specialist for UI tasks\n- Backend expert for API work\n- DevOps engineer for deployment\nYou analyze the context and files to determine your specialist role.",
  "customInstructions": "Analyze file extensions and project structure to determine your expertise focus:\n- *.tsx, *.css → Frontend mode\n- *.go, *.rs → Backend mode\n- Dockerfile, *.yaml → DevOps mode"
}
```

2. Progressive Enhancement Pattern:
```json
{
  "roleDefinition": "You are Roo, a security-focused code analyzer with progressive expertise levels",
  "customInstructions": "Apply security analysis in layers:\n1. Basic: Common vulnerabilities (OWASP Top 10)\n2. Advanced: Control flow and data handling\n3. Expert: Complex attack vectors and edge cases\n\nProgress through layers based on initial findings."
}
```

3. Context-Aware Behavior:
```json
{
  "roleDefinition": "You are Roo, a context-aware development assistant",
  "customInstructions": "Adapt behavior based on environment:\n- VS Code: Provide inline suggestions\n- Terminal: Focus on command explanations\n- Browser: Emphasize visual feedback\n\nDetect context from tool usage patterns."
}
```

## Limitations and Considerations

1. File Restrictions:
   - Tool groups with file restrictions must use valid regex patterns
   - Regex patterns are tested against full file paths
   - Invalid patterns will cause validation errors

2. Mode Conflicts:
   - Duplicate slugs are not allowed
   - Duplicate tool groups in a mode are not allowed
   - Mode names must be unique

3. Rule File Limitations:
   - Rule files must be in the project root
   - Only supports UTF-8 encoded text files
   - No support for binary files
   - Silent failure if rule files don't exist

4. Performance Considerations:
   - Rule files are read on every mode switch
   - Large rule files may impact performance
   - File watching may consume resources

## Best Practices

1. Mode Design:
   - Keep role definitions focused and specific
   - Use file restrictions to limit scope
   - Document mode capabilities clearly

2. Rule Organization:
   - Keep rules concise and clear
   - Use hierarchical structure
   - Separate concerns between modes

3. Tool Groups:
   - Grant minimum necessary permissions
   - Use file restrictions when possible
   - Document group usage in role definition

4. File Management:
   - Keep rule files in version control
   - Document rule changes
   - Review mode configurations regularly

## Advanced Usage Examples

### 1. Multi-Group Mode with Restrictions

```json
{
  "customModes": [
    {
      "slug": "frontend-dev",
      "name": "Frontend Developer",
      "roleDefinition": "Frontend development specialist focused on UI implementation",
      "groups": [
        "read",
        ["edit", { 
          "fileRegex": "\\.(js|jsx|ts|tsx|css|scss|html)$",
          "description": "Frontend source files"
        }],
        ["edit", {
          "fileRegex": "package\\.json$",
          "description": "Package management"
        }],
        "browser",
        "command"
      ]
    }
  ]
}
```

### 2. Specialized Testing Mode

```json
{
  "customModes": [
    {
      "slug": "tester",
      "name": "Test Engineer",
      "roleDefinition": "Specialized in test creation and maintenance",
      "groups": [
        "read",
        ["edit", {
          "fileRegex": "\\.(test|spec)\\.(js|ts)$",
          "description": "Test files only"
        }],
        "command"
      ],
      "customInstructions": "Always maintain test isolation and follow testing best practices"
    }
  ]
}
```

### 3. Documentation Mode

```json
{
  "customModes": [
    {
      "slug": "docs",
      "name": "Documentation Specialist",
      "roleDefinition": "Technical documentation and API documentation specialist",
      "groups": [
        "read",
        ["edit", {
          "fileRegex": "\\.(md|mdx|txt|doc)$",
          "description": "Documentation files"
        }],
        "browser"
      ],
      "customInstructions": "Follow technical writing best practices and maintain consistent terminology"
    }
  ]
}
```

## Integration Examples

### 1. Combining Rules and Mode Restrictions

```markdown
# .clinerules-docs
# Documentation Mode Rules

1. Documentation Standards:
   - Follow Google Technical Writing guidelines
   - Use consistent terminology
   - Include examples for complex concepts

2. File Organization:
   - Maintain clear hierarchy
   - Use meaningful file names
   - Keep related documentation together
```

Combined with the docs mode configuration above, this creates a specialized documentation environment with both behavioral rules and file access restrictions.

### 2. Cross-Mode Workflow

Example of how different modes can work together:

1. Architect Mode:
   - Creates design documents (.md)
   - Sets up project structure
   - Defines interfaces

2. Developer Mode:
   - Implements interfaces
   - Creates tests
   - Updates documentation

3. Documentation Mode:
   - Polishes documentation
   - Adds examples
   - Maintains API docs

Each mode has its own rules and restrictions while sharing common global rules from .clinerules.
