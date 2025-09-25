## Input.Select Tech interview 



### 🧠 1. Technical Decision-Making
**“Why did you build it this way?”**

## ✅ Be prepared to explain:
* Why you used virtualized rendering (@tanstack/react-virtual):
    * Large option sets → perf bottlenecks.
    * Virtualization ensures only visible rows are rendered, reducing DOM load.
* Why you built a custom useDropdownNavigation:
    * Full keyboard support: Arrow keys, focus tracking, accessibility.
    * Needed more control than generic focus libraries.
* Why you didn’t use a prebuilt library like Downshift, HeadlessUI, MUI, etc.:
    * Needed to conform to your design system.
    * Custom styling and behavior like Tree variant, accessibility spec, and ref forwarding weren't supported out-of-box.
### 🧩 Options you considered:
* Alternatives to useVirtualizer:
    * react-window or no virtualization at all.
* Accessibility via aria-activedescendant vs native focus shifting
    * Native focus shifting won for accessibility tool compatibility (especially NVDA).
* Portal vs inline dropdown:
    * If you chose inline, explain Z-index control and alignment challenges.
    * If portal, explain how you synced focus and scroll behavior.


### ⚙️ 2. Component Architecture & Extensibility
“How can this scale to future use cases?”
**Show that your component is:**
* **Modular:**
    * Internal logic split across: SelectOptionsList, useDropdownNavigation, DropdownShell, etc.
* **Composable:**
    * Can support flat lists, tree structures (variant="tree"), single-select or multi-select with minimal changes.
* *
    * You’ve planned for future needs like:
        * Search bar inside dropdown
        * Infinite scroll / async loading
        * Option groups (optgroup-style)
        * Support for aria-describedby in addition to aria-labelledby
* **Customizable through props:**
    * Controlled/uncontrolled behavior for value.
    * Optional id for accessibility.
    * toggleRef, data-testid for testability.


### 🔍 3. Performance & Accessibility
“Is this performant and accessible at scale?”

**Performance:**
* Virtualization reduces render cost on long option lists.
* React.memo and useCallback (if used) prevent unnecessary re-renders.
* Focus only set when necessary (avoiding layout thrashing).
* requestAnimationFrame used to delay focus in some experimental logic — shows perf awareness.
**Accessibility:**
* Follows WAI-ARIA 1.2 Combobox + Listbox pattern:
    * role="listbox", aria-labelledby, aria-activedescendant or manual focus.
* Keyboard navigation:
    * Arrow keys to move focus.
    * Tab, Enter, Esc behaviors handled.
* **Screen Reader behavior tested** (likely via NVDA or VoiceOver).
📌 You can reference that you’ve tested it using Cypress + Axe-core or manual screen reader passes.

### 🧪 4. Testing Strategy
“How do you know it works?”
**Cypress & RTL:**
* You used data-testids smartly (e.g., ${dataTestId}_option_${id}).
* Cypress tests:
    * Open/close behavior
    * Option selection
    * Keyboard nav (arrow keys, tab, escape)
    * Focus behavior assertions (cy.focused())
    * Tree variant interaction
🧪 Bonus:
* Mention percy visual regression if applicable.
* Mention any accessibility testing tooling (e.g., cypress-axe, Storybook a11y).

### 💡 5. Evolution of Your Code
“How did this evolve?”
You could describe a timeline like:

| Phase | Right Aligned |
|:-------------|--------------:|
| ✅ Initial Spike       | Build out the base dropdown using static rendering       |
| Item B       | $25.50        |



|Phase	|Focus|
|---|---|
✅ Initial Spike	Build out the base dropdown using static rendering
⚡ Iteration 1	Add keyboard nav (useDropdownNavigation)
🧠 Iteration 2	Add virtualization support
🧩 Iteration 3	Support Tree variant with nested groups
♿ Final Polish	Refine accessibility, testing, RTL support
Also highlight where you got feedback from UX/Accessibility team and how that shaped the final version.

🚧 6. Challenges Faced
“What was hard, and how did you solve it?”
Be honest here — some good talking points:
* Ensuring NVDA/JAWS read the right option with virtualization.
* Keeping focus behavior consistent between mouse and keyboard.
* Managing ref behavior inside virtualized lists (e.g., delayed rendering).
* Maintaining scroll alignment when updating currentIndex.

🔄 7. Potential Extensions
“How would you modify this to support X?”
Be prepared to answer how you’d:
* ✅ Add multi-select (checkboxes or chips)
* ✅ Support async option loading (infinite scroll, debounce)
* ✅ Add grouped options or nested trees
* ✅ Expose public API methods (e.g. .openDropdown(), .resetSelection())
If you've already structured SelectOptionsList and useDropdownNavigation generically, make sure to call that out as forward-thinking.

🗣️ Final Tips for Interview
* Practice talking aloud through one option → keyboard navigation → selection → focus.
* Use visual aids if allowed: Show the rendered DOM + virtualized items.
* Prepare a few good questions to ask them, like:
    * “How do you ensure accessibility at scale across your design system?”
    * “How do you approach performance testing for large data dropdowns?”




Gaps & improvements in SingleSelect.tsx

1) Accessibility correctness (combobox vs button+listbox)
Right now the root div has role="combobox", but the control is actually a button that opens a popup listbox. WAI-ARIA 1.2 patterns require specific linkages:
* If you keep a button + listbox pattern (not a text input combobox), use:
    * Button: aria-haspopup="listbox", aria-expanded, aria-controls="<popup id>"
    * Popup list: role="listbox" with active option role="option" and aria-selected="true"
    * Root element should not be role="combobox"; let the button carry the interactive semantics.
* If you truly want an ARIA combobox, that implies an input-like control with role="combobox", aria-controls, aria-expanded, aria-activedescendant, etc. That’s heavier and requires managing focus & typeahead.
Action:
* Remove role="combobox" from the outer div.
* On the toggle button add aria-haspopup="listbox" and ensure aria-controls targets the element with role="listbox" (your DropdownShell.Content).



2) ID collisions & id-optional bugs
You set the container div id={id} and render a hidden <input id={id}>. That’s an invalid duplicate DOM id. Also, if id is undefined, you compute things like ${id}_content which becomes "undefined_content".
Action:
* Make id required, or generate stable IDs internally (e.g., useId()).
* Use distinct IDs: container id, hidden input id, button id, popup id, list id.

Example:

const autoId = useId(); // React 18+
const baseId = id ?? `ss-${autoId}`;
const inputId = `${baseId}-input`;
const buttonId = `${baseId}-button`;
const listboxId = `${baseId}-listbox`;
const popupId = `${baseId}-popup`;


3) Tree search performance (flatten on every render)
flattenTree is recreated each render and you do a full flatten + find on every render. On large trees this is expensive and scales poorly.
Action:
* Build a map once with useMemo and do O(1) lookups.

const idToNode = useMemo(() => {
  if (variant !== 'tree') return null;
  const map = new Map<string, TreeNode>();
  const walk = (nodes: TreeNode[]) => nodes.forEach(n => {
    map.set(n.id, n);
    if (n.children) walk(n.children);
  });
  walk(options as TreeNode[]);
  return map;
}, [options, variant]);

const selectedOption = useMemo(() => {
  if (variant === 'tree') return idToNode?.get(value ?? '');
  const list = options as Option[];
  return list.find(o => o.id === value);
}, [idToNode, options, value, variant]);


6) Forward a ref & optional controlled open state
Consumers often need to focus the toggle or control the popup externally (forms, validation, or menus). Forwarding the ref unlocks this and an optional controlled open state is commonly requested.
Action:
* forwardRef<HTMLButtonElement, Props> and expose ref on the toggle.
* Optionally accept isOpen/onOpenChange to support controlled mode (keep uncontrolled as default).
7) ReadOnly & Disabled semantics
* If isReadOnly, the button is still tabbable and interactive visually. Add aria-readonly="true" and consider not rendering an interactive chevron, or at least set aria-hidden and remove hover affordances.
* If isDisabled, rely on disabled and add aria-disabled="true" on the wrapper for extra clarity.


11) Long list performance (future-proof)
You mentioned virtualization elsewhere—call it out in interview:
* For long flat lists, virtualize the SelectOptionsList.
* For trees, use windowed rendering per expanded branch.
* Memoize expensive props & item renderers; keep option objects referentially stable between renders to avoid re-diff in children.



Gaps & improvements in useDropdownNavigation.ts


1) Selected index doesn’t sync
The “sync to selectedIndex” logic lives in an effect that only depends on [options]. If selectedIndex changes (e.g., controlled parent), your hook won’t react. Also, you’re reading currentIndex in that effect but not listing it in deps.
Fix: Make a dedicated effect for external selection sync:

useEffect(() => {
  if (selectedIndex != null && selectedIndex >= 0) {
    setCurrentIndex(selectedIndex);
    moveFocusToIndex(selectedIndex);
  }
}, [selectedIndex]);


3) Event handlers re-bound on every render
You attach listeners inside an effect whose deps include currentIndex and options, which recreates handlers and re-binds events often. That’s unnecessary work and risks subtle bugs.
Fix: useCallback the handlers so they’re stable; then bind once, and refer to “latest state” via refs if needed.
