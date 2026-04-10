# Quick Notes (during development)

- Figma assets need to be trimmed to actual length, if there's parts that are hidden but still technically exist, the figma to code plugin picks it up still and messes with the creation.
- If the asset is a combination of multiple assets, copies of those assets need to be placed to the side to make those easier to design without having to move the objects in front of it out of the way.
- Assets like profile images are going to need to be exported and saved in the repo or it'll just call an api to get random portraits (will need to see if those are able to be used on the website or not).
- Some small icons are actually images in the figma and should instead be svgs (ex the chat with stars and the chat box footer).
- The figma to code plugin is unlimited free use (though I think is limited by the size of object) and works better than the AI just being given images or the figma mcp link.
- It would be nice to have a figma that we have access to drag things around or hide components to get cleaner screenshots of things for the AI (since it's easily distracted if there are other elements in the screenshot that you aren't trying to make).
- I think for development it's better to have these components in separate html (and css) files and then at the end, have the AI combine it into one document. I tried doing that with iFrames but those take much longer to load.
- It would be nice for when combining everything together if there was a UI to more easily adjust the placement of each thing. Have the AI try for a pass and then launch a gui to have real time adjustments instead of having to inspect, get each component and then having to delete and type in the new numbers.
- It would be easier to have an exact px by px aspect ratio for the entire 'image' to make things more presise 

---

# Process Retrospective: Figma-to-Static-HTML Hero Build

## Project Summary

We built a static HTML/CSS "hero image" -- a marketing composition of layered UI components -- entirely from Figma designs. The final deliverable is a single HTML file (`powerfully-easy-hero.html`) containing 10 inlined components (desktop UI, mobile chat UI, 3 data charts, 5 suggestion pills, panda SVG, leaf SVG) arranged with absolute positioning to recreate a complex Figma layout at pixel-level fidelity. A responsive wrapper (`hero-responsive-test.html`) scales the composition to any viewport width.

**Final file inventory:**

- `powerfully-easy-hero.html` -- the assembled hero (single inlined document, 1155x450 canvas)
- `hero-responsive-test.html` -- responsive iframe wrapper with `transform: scale()` + `aspect-ratio`
- `powerfully-easy-hero-iframes.html` -- archived iframe-based version (backup)
- `components/` -- 10 standalone component HTML files (desktop-ui, mobile-ui, 3 charts, 5 suggestion pills)
- `assets/` -- 8 SVG files (panda, leaf, chat icons, like/dislike thumbs, etc.)
- `hero-images/` -- reference PNGs from Figma

## What Worked

### 1. Component-first architecture

Building each UI piece as its own standalone HTML file was the single most important decision. Each component could be developed, previewed, and iterated on independently without worrying about how it fit into the larger whole. When the time came to assemble, the components were already correct and just needed positioning.

**Components built independently:**

- `desktop-ui.html` (Employee Community + My Stuff panel)
- `mobile-ui.html` (Ask BambooHR chat interface)
- `headcount-by-location.html` (donut chart)
- `turnover-by-type.html` (bar chart)
- `new-hires-by-department.html` (horizontal bar chart)
- 5 suggestion pill components in `components/suggestions/`

### 2. Figma-to-code plugin as the starting point

Using the Figma plugin's code export for initial HTML/CSS markup was significantly faster than having the AI write everything from screenshots alone. The plugin captured exact dimensions, colors, font sizes, border-radii, and shadow values that would have taken many correction cycles to get right from visual inspection. The AI's job became refining and fixing the plugin output rather than writing from scratch.

### 3. Providing multiple reference formats simultaneously

The most efficient component builds happened when the AI received all of the following at once:

- A close-up screenshot of the target component
- The Figma plugin's exported code
- Specific measurements from Figma (widths, heights, spacing)
- A screenshot of the component in context (in the full hero)

When only a screenshot was provided without measurements, the AI had to guess at sizes and needed more correction rounds.

### 4. Inlining components into one document

Replacing all iframes with inline HTML/CSS in a single file solved multiple problems at once:

- Eliminated 10 separate HTTP requests for component HTML files
- Eliminated duplicate Google Fonts and Font Awesome loads (each iframe loaded them independently)
- Eliminated iframe clipping bugs where box-shadows were cut off
- Reduced total page load time from several seconds to near-instant
- Made the file self-contained and portable

### 5. Keeping a backup before major refactors

Before inlining, we copied the iframe-based version to `powerfully-easy-hero-iframes.html`. This meant we could experiment freely with the inlined version knowing we could always go back. This is a small thing but it removed the anxiety of a big structural change.

### 6. Responsive wrapper with transform: scale()

The approach of embedding the fixed-size hero in an iframe, then using JavaScript to compute `container.clientWidth / canvasWidth` and apply `transform: scale()`, was elegant. Combined with `aspect-ratio` on the container, the composition behaves exactly like a responsive image -- it scales uniformly, nothing reflows or shifts, and it works at any viewport width.

### 7. Embed mode pattern

Each component had a dual-mode design: standalone mode (dark background, centered, with browser preview affordances) and embed mode (transparent background, no centering, no padding) toggled via a `?embed` URL parameter and `html.hero-embed` CSS class. This was useful during the iframe phase and could be useful again if components need to be shown independently for documentation or testing.

## What Didn't Work / Pain Points

### 1. The iframe phase was a costly detour

We spent significant time on an iframe-based assembly approach that ultimately had to be replaced. Key problems:

- **Shadow clipping**: Every component with a `box-shadow` that extended beyond its element boundaries was clipped by the iframe. This required adding padding hacks to each component's embed mode and adjusting every iframe's size and position to compensate. Multiple rounds of this for multiple components.
- **Font duplication**: Each iframe independently loaded Google Fonts and Font Awesome, causing 10x the HTTP requests and visible FOIT (flash of invisible text) as fonts loaded asynchronously across iframes.
- **Debugging difficulty**: When something looked wrong in the assembled hero, it was hard to tell if the issue was in the component's CSS, the iframe sizing, the embed mode padding, or the parent page's positioning.

In hindsight, going straight to inlined HTML from the start would have saved substantial time.

### 2. Negative-position overflow caused invisible content

The leaf SVG was positioned at `top: -30px` to extend above the panda's head. This looked fine when the hero was viewed directly (since `.hero-canvas` had `overflow: visible`), but when embedded in an iframe for the responsive test, the content above y=0 was invisible. Fixing this required adding 30px to the canvas height and shifting every single positioned element's `top` value down by 30px -- a tedious 13-property change that touched every component.

**Lesson**: Keep all content within positive coordinates. Add canvas padding up front rather than using negative offsets.

### 3. Positional tuning was the biggest time sink

Adjusting the `left`, `top`, `width`, `height`, and `z-index` of 12+ absolutely positioned elements to match a reference image was by far the most time-consuming part. Each adjustment required: edit a CSS value -> save -> check the browser -> compare to reference -> decide what to change next. There's no good tooling for this -- you can't drag things around and see coordinates update.

The AI's first attempt at positioning from the reference image was significantly off. It took many rounds of human feedback ("move this left," "this is overlapping that," "the z-index is wrong here") to converge. Even with Playwright screenshots for the AI to compare against, the feedback loop was slow.

### 4. Canvas dimensions required multiple resizes

The canvas went through four sizes: `1165x446` -> `2329x891` (2x scale, reverted) -> `1190x476` -> `1155x450`. Each resize required updating the hero file, the responsive test file (aspect-ratio, iframe dimensions, JS scale calculation), and sometimes component positions. Having the correct bounding box from the start would have avoided all of these.

### 5. External edits caused confusion

At one point, two suggestion pill positions were changed outside the AI session (possibly a manual edit or another tool). The AI didn't know about the change and the user reported pills being "way over to the left." Debugging this required diffing the expected vs. actual CSS values. In a collaborative workflow, this is a real risk -- the AI's mental model of the file state can desync.

### 6. Box-shadow bounding was repeatedly underestimated

Many components had directional box-shadows (e.g., `-4.3px 4.3px 0 #599d15` -- shadow goes left and down). These shadows extended outside the component's box but were invisible when clipped by iframes or canvas boundaries. This came up with:

- Mobile UI left border clipping
- Desktop UI left border clipping
- Turnover chart left edge clipping
- New Hires chart right edge clipping

Each time, the fix was to add a few pixels of space. A systematic approach would have been to calculate the maximum shadow extent across all components and build that into the canvas padding from the start.

### 7. Live-server was set up but never actually used

We installed `live-server` and added an `npm run dev` script specifically for hot-reload during the visual tuning phase. However, the old `preview-server.mjs` (a basic static server on port 5173 with no file-watching) was already running in a terminal, and we kept using that instead of switching to `live-server` on port 5174. The result: the entire positional tuning phase was done with manual save-and-reload, which is exactly the slow workflow live-server was supposed to fix.

**Lesson**: Setting up a tool isn't enough -- you have to actually confirm it's running and that you're using it. The AI should have verified the user was viewing pages on the live-server port, not just that the package was installed. Next time, kill the old server, start live-server, and open the live-server URL in the browser before moving on.

### 8. Some components fought the HTML/CSS box model unnecessarily

The first chart component (headcount by location, a donut chart) was difficult to get right as styled HTML divs. Precise arcs, exact bar widths, and pixel-aligned shapes don't map well to `div` + `border-radius` hacks. It worked much better once we switched to inline SVG, which is purpose-built for geometric shapes and precise positioning. The AI should proactively flag when a component looks like it would be more natural as SVG rather than styled markup.

### 9. AI struggles with pixel-precise layout from images alone

When given only a reference image, the AI's ability to estimate absolute pixel positions is poor. It can get the general arrangement right (what's on the left, what overlaps what) but specific coordinates are usually off by 20-100px. The AI is much more effective when given:

- Exact coordinates from Figma's inspector
- Component widths/heights as numbers
- The canvas dimensions as explicit values

## What Would Make the Process Easier

### For the human (Figma prep)

1. **Export the full composition's bounding box** from Figma -- the exact pixel dimensions of the hero frame. This eliminates canvas-sizing iteration.
2. **Export each component's position** within the composition (x, y coordinates relative to the hero frame, plus width and height). This data is available in Figma's inspector and would make the assembly phase near-instant instead of hours of tuning.
3. **Trim Figma assets** to their visible bounds before exporting code. Hidden overflow, off-screen elements, and invisible layers all produce garbage markup.
4. **Isolate components in Figma** before screenshotting. The AI gets confused by surrounding elements in a screenshot. Move the component to a clean artboard or hide adjacent layers.
5. **Use SVGs for small icons** instead of raster images. Check that icons in the Figma are vectors, not embedded PNGs.
6. **Save profile images / photos** as real files in the repo rather than relying on external URLs that may not render in static contexts.

### For the AI workflow

1. **Skip the iframe phase entirely**. Inline from the start. Build each component as a standalone file for development, but when assembling, copy each component's inner HTML and CSS directly into the hero file.
2. **Calculate the canvas bounding box before building**. Take the max extent of all components (including shadows) and set canvas dimensions once.
3. **Namespace all CSS classes** during the inline phase. Prefix component classes (e.g., `.desktop-`*, `.mobile-*`, `.chart-*`) to avoid collisions when everything lives in one stylesheet.
4. **Keep all positions in positive coordinates**. Never use negative `top` or `left`. Add canvas padding if an element needs to extend beyond the visual origin.
5. **Pre-plan the z-index stack** as a numbered list before placing any elements. This avoids ad-hoc z-index escalation and overlapping surprises.
6. **Set up live-server immediately and verify it's actually being used.** Install it, start it, kill any competing static servers, and confirm the browser is pointed at the live-server port. Don't just install the package and assume it's working -- the hot-reload feedback loop is essential for the visual tuning phase, and this time we installed it but never actually benefited from it.
7. **Build a position-tuning overlay**. A small JavaScript snippet that makes elements draggable and logs their coordinates on drop would replace hours of manual CSS editing. Even a basic implementation would pay for itself immediately.

---

# Reusable AI Prompt: Figma-to-Static-HTML Hero Builder

Copy everything below the line into a new AI session to walk through the full process.

---

## PROMPT START

You are helping me build a static HTML/CSS "hero image" -- a marketing composition made of multiple layered UI components. The final output is a single, self-contained HTML + CSS file pair where every component is inlined (no iframes) and arranged with absolute positioning to pixel-match a Figma design. It should also have a responsive wrapper that scales the composition uniformly to fit any container width.

**This is a static image, not an interactive UI.** Even though the components look like real UI (buttons, text inputs, chat messages), the final output must behave like a flat image. Nothing should be focusable, clickable, selectable, or editable. No element should show a cursor change on hover. Unless I say otherwise, treat the entire composition as purely visual.

### What I will provide you

Before we start, I need to gather and give you the following. Ask me for anything I haven't provided.

**Required:**

1. A reference image (PNG/screenshot) of the full hero composition showing the final layout
2. The exact pixel dimensions of the hero composition frame from Figma (width x height)
3. For each component in the composition:
  - A close-up screenshot of the component
  - The Figma-to-code plugin's exported HTML/CSS (or the raw code I've written)
  - The component's position within the hero frame (x, y from Figma's inspector)
  - The component's width and height in pixels
4. Any SVG assets (icons, illustrations) as raw SVG code or files

**Nice to have (only if not already captured in the Figma plugin export):**

- Font names and weights (e.g., Inter 400/500/600/700) -- the plugin export usually includes these, so only ask if missing
- Key color values (background, brand, shadow colors) -- same, the plugin export usually captures them
- A z-index stacking order (which components are in front of which)
- The box-shadow values for each component (so we can account for shadow overflow)
- Profile images or other raster assets as files in the repo

### How we will work

#### Phase 1: Project setup

1. Create a project folder with this structure:
  ```
   project/
     assets/          (SVGs, images)
     components/      (HTML + CSS file pairs for each piece)
     hero-images/     (reference PNGs from Figma)
     package.json     (with live-server as a dev dependency)
  ```
2. Set up `live-server` for hot reload immediately: `npm init -y && npm install --save-dev live-server`, add a `"dev": "live-server --port=5174"` script. **Start it, kill any other static servers that might be running, and confirm I'm viewing pages at the live-server URL (http://localhost:5174).** Do not proceed until hot reload is verified working -- this saves enormous time during visual tuning.
3. Create an `index.html` with links to each component for easy navigation during development.

#### Phase 2: Build components individually

For each component, one at a time:

1. Create a separate HTML file and CSS file in `components/` (e.g., `components/desktop-ui.html` + `components/desktop-ui.css`). The HTML file links to its CSS file via `<link rel="stylesheet">`. Keep styles out of the HTML.
2. Start from the Figma plugin's code export if available, otherwise build from the screenshot + dimensions I provide. Extract font names, colors, and other tokens directly from the plugin export -- don't ask me for values the export already contains.
3. The HTML file should be a complete document that links to its own CSS file plus any font/icon CDN links. The CSS file contains all component styles.
4. The standalone file should have a dark background (`#1a1a1a`) and center the component for easy preview.
5. **Namespace all CSS classes** with a component prefix (e.g., `.desktop-*`, `.mobile-*`, `.chart-*`) to avoid collisions when merging later.
6. **Evaluate whether the component is better built as an SVG.** Charts, graphs, and data visualizations with precise shapes and alignments are often easier and more accurate as inline SVG than as styled HTML divs. If a component involves bars, arcs, precise geometric shapes, or is fighting the box model, suggest SVG and explain the trade-off. Build whichever approach I agree to.
7. Show me the result. I will provide corrections (sizing, spacing, colors, etc.) and you iterate until it matches.

**Do not move to the next component until I confirm the current one is correct.**

#### Phase 3: Assemble the hero (single inlined document)

Once all components are done:

1. Create the hero HTML file (e.g., `powerfully-easy-hero.html`) and a corresponding CSS file (e.g., `powerfully-easy-hero.css`). The HTML links to the CSS file.
2. Merge all component CSS files into the single hero CSS file. Resolve any class name collisions.
3. Load fonts (Google Fonts, Font Awesome) exactly once in the HTML `<head>`.
4. Create a `.hero-canvas` div with the exact Figma frame dimensions:
  ```css
   .hero-canvas {
     position: relative;
     width: [FIGMA_WIDTH]px;
     height: [FIGMA_HEIGHT]px;
     overflow: visible;
   }
  ```
5. For each component, copy its inner HTML into a `<div class="hero-layer hero-[name]">` inside the canvas.
6. Position each layer using the Figma coordinates I provided:
  ```css
   .hero-[name] { left: [X]px; top: [Y]px; z-index: [Z]; }
  ```
7. Update all relative asset paths (e.g., `../assets/foo.svg` becomes `assets/foo.svg`).

**Critical rules for assembly:**

- **All `top` and `left` values must be positive.** If any element needs to extend above or left of the canvas origin, increase the canvas dimensions and shift all elements to compensate. Never use negative positions.
- **Account for box-shadows in the canvas size.** If a component has a shadow that extends 5px left, make sure there's at least 5px of space between the component's left edge and the canvas left edge. Do this for all four sides.
- **Set `overflow: visible` on the canvas and all layers** so shadows and outlines are not clipped.
- **Do not use iframes.** Inline everything. Iframes cause shadow clipping, duplicate font loading, and slower page loads.
- **Make the entire composition non-interactive.** Apply `pointer-events: none; user-select: none;` to the `.hero-canvas`. Add `tabindex="-1"` to any `<input>`, `<button>`, `<a>`, or `<textarea>` elements so nothing is focusable via keyboard. The composition should behave like an image -- no hover states, no text selection, no tab stops, no clickable areas.

#### Phase 4: Visual tuning

1. Compare the rendered hero to the reference image I provided.
2. If you have access to Playwright or a screenshot tool, take a screenshot and compare side-by-side.
3. Adjust positions, z-indexes, and sizes iteratively. Ask me for feedback if unsure.
4. Common issues to watch for:
  - Elements overlapping in the wrong order (z-index)
  - Box-shadows being clipped at canvas edges
  - Text rendering differences between Figma and the browser
  - Components shifted a few pixels from their target position

#### Phase 5: Responsive wrapper

Once the hero looks correct at its native size:

1. Create `hero-responsive-test.html` with this structure:
  ```html
   <style>
     body { padding: 24px; }
     .hero-container {
       width: 100%;
       aspect-ratio: [CANVAS_WIDTH] / [CANVAS_HEIGHT];
       overflow: hidden;
       outline: 2px solid #555;
     }
     .hero-container iframe {
       display: block;
       width: [CANVAS_WIDTH]px;
       height: [CANVAS_HEIGHT]px;
       border: none;
       transform-origin: top left;
     }
   </style>
   <div class="hero-container">
     <iframe src="powerfully-easy-hero.html"
             width="[CANVAS_WIDTH]" height="[CANVAS_HEIGHT]"
             scrolling="no"></iframe>
   </div>
   <script>
     function scale() {
       const c = document.querySelector('.hero-container');
       const f = c.querySelector('iframe');
       f.style.transform = 'scale(' + (c.clientWidth / [CANVAS_WIDTH]) + ')';
     }
     window.addEventListener('resize', scale);
     window.addEventListener('DOMContentLoaded', scale);
   </script>
  ```
2. Test at various viewport widths. The hero should scale uniformly like an image -- nothing should reflow or shift, just get proportionally smaller/larger.
3. Check that no content is clipped at the edges. If it is, increase the canvas size (not move the content), then update the responsive wrapper's dimensions to match.

### Pitfalls to avoid

1. **Never use iframes for assembly.** They clip shadows, duplicate font loads, and add complexity. Inline everything.
2. **Never use negative `top` or `left` values.** They cause invisible overflow when the hero is embedded in anything with `overflow: hidden`.
3. **Don't guess positions from screenshots.** Always ask me for the exact Figma coordinates. Screenshot-based positioning is off by 20-100px and wastes iteration time.
4. **Don't forget shadow extent in canvas sizing.** A `box-shadow: -4px 4px 0 green` means the element needs at least 4px of clearance on its left and bottom edges.
5. **Don't load fonts more than once.** One Google Fonts `<link>` and one Font Awesome `<link>` at the top of the hero file. That's it.
6. **Don't just install live-server -- verify it's actually running and being used.** Kill any other static servers, start live-server, and confirm the browser is pointed at the correct port. The visual tuning phase is unbearable without hot reload, and simply installing the package without switching to it wastes the effort.
7. **Don't edit positions outside this session without telling me.** If you make changes I don't know about, I can't debug visual regressions.

### When you're done

The final deliverables should be:

- `powerfully-easy-hero.html` + `powerfully-easy-hero.css` (or equivalent) -- the assembled hero at native resolution
- `hero-responsive-test.html` -- responsive wrapper that scales it to any viewport width
- `components/` directory with standalone HTML + CSS file pairs preserved for future editing
- `assets/` directory with all SVGs and images

## PROMPT END

