---
name: frontend-design
description: Create distinctive, production-grade frontend interfaces with high design quality. Use this skill when the user asks to build web components, pages, or applications. Generates creative, polished code that avoids generic AI aesthetics.
---

This skill guides creation of distinctive, production-grade frontend interfaces that avoid generic "AI slop" aesthetics. Implement real working code with exceptional attention to aesthetic details and creative choices.

The user provides frontend requirements: a component, page, application, or interface to build. They may include context about the purpose, audience, or technical constraints.

## Design Thinking

Before coding, understand the context and commit to a BOLD aesthetic direction:
- **Purpose**: What problem does this interface solve? Who uses it?
- **Tone**: Pick an extreme: brutally minimal, maximalist chaos, retro-futuristic, organic/natural, luxury/refined, playful/toy-like, editorial/magazine, brutalist/raw, art deco/geometric, soft/pastel, industrial/utilitarian, etc. Use these for inspiration but design one that is true to the aesthetic direction.
- **Constraints**: Technical requirements (framework, performance, accessibility).
- **Differentiation**: What makes this UNFORGETTABLE? What's the one thing someone will remember?

**CRITICAL**: Choose a clear conceptual direction and execute it with precision. Bold maximalism and refined minimalism both work — the key is intentionality, not intensity.

Then implement working code (HTML/CSS/JS, React, Vue, etc.) that is:
- Production-grade and functional
- Visually striking and memorable
- Cohesive with a clear aesthetic point-of-view
- Meticulously refined in every detail

## Frontend Aesthetics Guidelines

Focus on:
- **Typography**: Choose fonts that are beautiful, unique, and interesting. Avoid generic fonts like Arial and Inter; opt for distinctive choices that elevate aesthetics — unexpected, characterful font pairs (display + body).
- **Color & Theme**: Commit to a cohesive aesthetic. Use CSS variables for consistency. Dominant colors with sharp accents outperform timid, evenly-distributed palettes.
- **Motion**: Use animations for effects and micro-interactions. Prioritize CSS-only solutions for HTML. Use Motion library for React when available. Focus on high-impact moments: one well-orchestrated page load with staggered reveals creates more delight than scattered micro-interactions. Use scroll-triggering and hover states that surprise.
- **Spatial Composition**: Unexpected layouts. Asymmetry. Overlap. Diagonal flow. Grid-breaking elements. Generous negative space OR controlled density.
- **Backgrounds & Visual Details**: Create atmosphere and depth. Apply creative forms like gradient meshes, noise textures, geometric patterns, layered transparencies, dramatic shadows, decorative borders, and grain overlays.

NEVER use generic AI-generated aesthetics: overused font families (Inter, Roboto, Arial, system fonts), clichéd color schemes (purple gradients on white), predictable layouts, or cookie-cutter design that lacks context-specific character. No design should be the same — vary themes, fonts, and aesthetics across generations. NEVER converge on common choices (Space Grotesk, for example).

**IMPORTANT**: Match implementation complexity to the aesthetic vision. Maximalist designs need elaborate code with extensive animations. Minimalist designs need restraint, precision, and careful attention to spacing and subtle details.

## Formhause Design Philosophy

Apply the above while preserving the existing Formhause theming. Keep any shadcn components or existing styles. Changes should be **subtly powerful** — don't drastically alter the current look unless explicitly prompted. Spacing should be pleasant.

## Code Generation Guidelines

- Some users may be on older browsers — include fallbacks where needed.
- The application uses light and dark mode with `dark:` Tailwind classes — account for both in any generated styles.
- **Depth cues differ by mode**: In light mode, use shadows (box-shadow, drop-shadow) to convey elevation and distance. In dark mode, shadows are invisible against dark backgrounds — instead use lighter surface colors to bring elements forward and darker colors to push them back. A card in light mode might use `shadow-md`; the same card in dark mode should use a lighter background (e.g. `dark:bg-zinc-800`) against a darker page (e.g. `dark:bg-zinc-950`) to achieve the same visual separation. **Shadows should be subtle** — they convey depth, not draw attention. Prefer `shadow-sm` or `shadow-md` over `shadow-lg`/`shadow-xl`. If a shadow is the first thing a user notices, it's too heavy.
- **Avoid semantic color defaults** — never use generic CSS variable classes like `bg-card`, `bg-background`, `bg-muted`, or `text-foreground` for new UI. These resolve to bland defaults that lack visual intention. Instead, use explicit zinc scale values (e.g. `bg-zinc-50`, `dark:bg-zinc-800`) to control exactly how surfaces look in both modes. Semantic classes are fine for existing shadcn components (Button, Card, etc.) but when building custom layouts and sections, pick specific shades from the zinc palette for precise control.
- Avoid overly complicated animations.

Remember: Claude is capable of extraordinary creative work. Don't hold back — show what can truly be created when committing fully to a distinctive vision.
