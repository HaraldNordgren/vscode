# VS Code Theming Architecture

This document explains how theming works in VS Code, covering the code involved, data formats, and mechanisms for consumers.

## Overview

VS Code has **three types of themes**:
1. **Color Themes** - Define colors for the UI and syntax highlighting
2. **File Icon Themes** - Define icons for files/folders in the explorer
3. **Product Icon Themes** - Define icons used in the UI (codicons)

---

## 1. Code Architecture

### Core Services

| Service | Location | Purpose |
|---------|----------|---------|
| `IThemeService` | [platform/theme/common/themeService.ts](src/vs/platform/theme/common/themeService.ts) | Base interface for theme access |
| `IWorkbenchThemeService` | [workbench/services/themes/common/workbenchThemeService.ts](src/vs/workbench/services/themes/common/workbenchThemeService.ts) | Extended interface with theme switching |
| `WorkbenchThemeService` | [workbench/services/themes/browser/workbenchThemeService.ts](src/vs/workbench/services/themes/browser/workbenchThemeService.ts) | Main implementation |

### Theme Data Classes

| Class | Location | Purpose |
|-------|----------|---------|
| `ColorThemeData` | [workbench/services/themes/common/colorThemeData.ts](src/vs/workbench/services/themes/common/colorThemeData.ts) | Loads & manages color theme data |
| `FileIconThemeData` | [workbench/services/themes/browser/fileIconThemeData.ts](src/vs/workbench/services/themes/browser/fileIconThemeData.ts) | File icon theme data |
| `ProductIconThemeData` | [workbench/services/themes/browser/productIconThemeData.ts](src/vs/workbench/services/themes/browser/productIconThemeData.ts) | Product icon theme data |

### Registry & Extension Points

| Component | Location | Purpose |
|-----------|----------|---------|
| `ColorRegistry` | [platform/theme/common/colorUtils.ts](src/vs/platform/theme/common/colorUtils.ts) | Central registry for all color identifiers |
| `ThemeRegistry` | [workbench/services/themes/common/themeExtensionPoints.ts](src/vs/workbench/services/themes/common/themeExtensionPoints.ts) | Manages theme contributions from extensions |
| Extension Points | [workbench/services/themes/common/themeExtensionPoints.ts](src/vs/workbench/services/themes/common/themeExtensionPoints.ts) | `themes`, `iconThemes`, `productIconThemes` |

---

## 2. Input Data (Extension JSON Files)

### Color Theme Extension Contribution (`package.json`)

```json
{
  "contributes": {
    "themes": [{
      "id": "my-theme",
      "label": "My Theme",
      "uiTheme": "vs-dark",  // vs, vs-dark, hc-black, hc-light
      "path": "./themes/my-theme-color-theme.json"
    }]
  }
}
```

### Color Theme File Format (`.json` or `.tmTheme`)

```json
{
  "name": "My Theme",
  "colors": {
    "editor.background": "#1e1e1e",
    "editor.foreground": "#d4d4d4",
    "activityBar.background": "#333333"
    // ... 500+ possible color keys
  },
  "tokenColors": [
    {
      "scope": ["comment", "punctuation.definition.comment"],
      "settings": {
        "foreground": "#6A9955",
        "fontStyle": "italic"
      }
    }
  ],
  "semanticHighlighting": true,
  "semanticTokenColors": {
    "variable.readonly": "#4FC1FF"
  }
}
```

### File Icon Theme File Format

```json
{
  "iconDefinitions": {
    "_file": { "iconPath": "./icons/file.svg" },
    "_folder": { "iconPath": "./icons/folder.svg" }
  },
  "file": "_file",
  "folder": "_folder",
  "fileExtensions": {
    "ts": "_typescript"
  }
}
```

---

## 3. Output Data (Generated CSS)

The theming system generates **CSS stylesheets** that are injected into the DOM:

### CSS Variable Generation

Colors are exposed as CSS variables on `.monaco-workbench`:

```css
.monaco-workbench {
  --vscode-editor-background: #1e1e1e;
  --vscode-editor-foreground: #d4d4d4;
  --vscode-activityBar-background: #333333;
  /* ... all registered colors */
}
```

The conversion follows this pattern:
- Color ID: `editor.background`
- CSS Variable: `--vscode-editor-background`

### Style Sheet Application

Three separate stylesheets are managed:

```typescript
const colorThemeRulesClassName = 'contributedColorTheme';
const fileIconThemeRulesClassName = 'contributedFileIconTheme';
const productIconThemeRulesClassName = 'contributedProductIconTheme';
```

Applied via `_applyRules()`:
```typescript
function _applyRules(styleSheetContent: string, rulesClassName: string) {
  const themeStyles = mainWindow.document.head.getElementsByClassName(rulesClassName);
  if (themeStyles.length === 0) {
    const elStyle = createStyleSheet();
    elStyle.className = rulesClassName;
    elStyle.textContent = styleSheetContent;
  } else {
    themeStyles[0].textContent = styleSheetContent;
  }
}
```

---

## 4. Consumer Mechanisms

### 4.1 Theming Participants (CSS Rules)

Components can register to generate CSS rules when theme changes:

```typescript
import { registerThemingParticipant } from 'vs/platform/theme/common/themeService';

registerThemingParticipant((theme, collector) => {
  const background = theme.getColor(editorBackground);
  if (background) {
    collector.addRule(`.my-component { background-color: ${background}; }`);
  }
});
```

### 4.2 Direct Color Access via CSS Variables

Use CSS variables in stylesheets:

```css
.my-element {
  background-color: var(--vscode-editor-background);
  color: var(--vscode-editor-foreground);
}
```

### 4.3 Programmatic Color Access

Inject `IThemeService` and query colors:

```typescript
class MyComponent {
  constructor(@IThemeService private readonly themeService: IThemeService) {
    const theme = this.themeService.getColorTheme();
    const color = theme.getColor(editorBackground);
  }
}
```

### 4.4 Themable Base Class

Extend `Themable` for automatic theme updates:

```typescript
import { Themable } from 'vs/platform/theme/common/themeService';

class MyWidget extends Themable {
  constructor(@IThemeService themeService: IThemeService) {
    super(themeService);
  }

  updateStyles(): void {
    // Called automatically on theme change
    const color = this.getColor(editorBackground);
  }
}
```

### 4.5 Registering New Colors

Register colors in the color registry:

```typescript
import { registerColor, transparent } from 'vs/platform/theme/common/colorUtils';

export const myWidgetBackground = registerColor(
  'myWidget.background',
  {
    dark: '#252526',
    light: '#F3F3F3',
    hcDark: '#000000',
    hcLight: '#FFFFFF'
  },
  'Background color for my widget'
);
```

Colors are organized in files under:
- [platform/theme/common/colors/](src/vs/platform/theme/common/colors/) - Base colors by category

### 4.6 Theme Change Events

Subscribe to theme changes:

```typescript
themeService.onDidColorThemeChange((theme) => {
  // React to theme change
});

themeService.onDidFileIconThemeChange((theme) => { });
themeService.onDidProductIconThemeChange((theme) => { });
```

---

## 5. Color Resolution Flow

```
Extension Theme JSON
        ↓
ColorThemeData.load()
        ↓
    ┌───────────────────────┐
    │ Theme Colors (colors) │
    │ Token Colors          │
    │ Semantic Token Rules  │
    └───────────────────────┘
        ↓
User Customizations Applied
(workbench.colorCustomizations)
        ↓
ColorRegistry.resolveDefaultColor()
        ↓
CSS Variables Generated
        ↓
DOM <style> Element Updated
```

---

## 6. Key Settings

| Setting | Purpose |
|---------|---------|
| `workbench.colorTheme` | Active color theme ID |
| `workbench.iconTheme` | Active file icon theme ID |
| `workbench.productIconTheme` | Active product icon theme ID |
| `workbench.colorCustomizations` | User color overrides |
| `editor.tokenColorCustomizations` | User token color overrides |
| `editor.semanticTokenColorCustomizations` | Semantic token overrides |
| `window.autoDetectColorScheme` | Auto-switch light/dark |

---

## 7. Theme Type Selectors

CSS classes applied to root element based on theme type:

| Class | Theme Type |
|-------|------------|
| `vs` | Light |
| `vs-dark` | Dark |
| `hc-black` | High Contrast Dark |
| `hc-light` | High Contrast Light |

Components can use these for conditional styling:

```css
.vs .my-element { /* light theme styles */ }
.vs-dark .my-element { /* dark theme styles */ }
.hc-black .my-element { /* high contrast dark */ }
```

---

## Summary

| Aspect | Description |
|--------|-------------|
| **Input** | Theme JSON files contributed by extensions via `package.json` |
| **Processing** | `WorkbenchThemeService` loads, merges user customizations, resolves colors |
| **Output** | CSS variables and rules injected into DOM `<style>` elements |
| **Consumers** | CSS variables, `registerThemingParticipant`, `IThemeService`, `Themable` base class |
