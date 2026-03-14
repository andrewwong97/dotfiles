---
description: Storybook conventions for frontend components
paths: ["frontend/**/*.ts", "frontend/**/*.tsx"]
---

# Storybook Rules

When building any new component or page in `frontend/`, build a corresponding `*.stories.tsx` file. **Always place new stories in `frontend/src/stories/`** — do not co-locate them next to the component. Include at least one baseline story per file (`Default` is common).

## Config & Conventions

- Storybook config lives in `frontend/.storybook/`.
- Stories are discovered from `frontend/src/**/*.stories.@(js|jsx|mjs|ts|tsx)`.
- All stories must be written in TypeScript (`.stories.tsx`). Explicitly type all parameters, decorator arguments, and destructured inputs — no implicit `any`.
- Import types from `@storybook/react-vite` (not `@storybook/react`).
- For interaction tests in `play` functions, use `within` from `@testing-library/react` and `userEvent` from `@testing-library/user-event`.

## Decorators

Pages and components that depend on React context or routing must be wrapped in decorators:

- **Routing** (`useNavigate`, `useLocation`, etc.) — wrap with `MemoryRouter` or `BrowserRouter` from `react-router-dom` (both are used in this repo).
- **Theme** (`useTheme`) — wrap with `ThemeProvider` from `@/contexts/ThemeContext`.
- Type the `Story` parameter in decorators explicitly, e.g. `(Story: React.FC) => (...)`.

## Play Functions

Use `play` functions for interactive stories. The callback receives `{ canvasElement }` (a raw DOM node):

```ts
play: async ({ canvasElement }: { canvasElement: HTMLElement }) => {
  const canvas = within(canvasElement);
  await userEvent.click(canvas.getByText('Sign Up'));
},
```
