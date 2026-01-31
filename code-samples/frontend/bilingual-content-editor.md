# Bilingual Content Editor

## Context

**Feature:** Rich Content Editing with RTL/LTR Support  
**Complexity:** High  
**Technologies Used:** React, Framer Motion, CSS Logical Properties

**The Challenge:**
Build a content editor that seamlessly handles both Arabic (RTL) and English (LTR) text with proper bidirectional layout, integrated with drag-and-drop reordering.


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## The Problem

Most form libraries assume single-language, left-to-right text:

```
Standard Form Field:
┌─────────────────────────────┐
│ Enter text here...          │
└─────────────────────────────┘
                               ← Always LTR

Bilingual Requirement:
┌─────────────────────────────┐
│                  ...أدخل النص │ ← RTL
└─────────────────────────────┘
┌─────────────────────────────┐
│ Enter text here...          │ ← LTR
└─────────────────────────────┘
```

**Requirements:**
- Side-by-side Arabic (RTL) and English (LTR) editing
- Proper text direction per language
- Visual alignment respecting text direction
- Drag-and-drop for content blocks

**Constraints:**
- Cannot use `dir="auto"` (not reliable for mixed content)
- CSS must handle both directions
- Animations must be direction-aware
- Form state must track both languages


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## The Approach

**"Direction as a First-Class Concern"** - Every component explicitly knows its text direction.

### Why This Pattern?

Instead of trying to auto-detect direction, we:
1. Explicitly pass `direction` to each component
2. Use CSS Logical Properties (`inline-start`, `inline-end`)
3. Apply appropriate font stacks per language

### Alternatives Considered

1. **Two Separate Forms (AR/EN tabs)**
   - Pros: Simpler implementation
   - Cons: Can't see both versions, easy to miss translations

2. **Auto RTL Detection (dir="auto")**
   - Pros: No explicit handling
   - Cons: Unreliable for mixed content, edge cases

3. **Explicit Direction per Field** ✅
   - Pros: Reliable, predictable, full control
   - Trade-offs: More verbose, requires design consistency


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Implementation (Simplified)

### Bilingual Text Field

```typescript
/**
 * DISCLAIMER: This is a simplified version for demonstration purposes.
 * The actual implementation includes:
 * - Validation per language
 * - Auto-save drafts
 * - Character count limits
 */

interface BilingualTextFieldProps {
  arValue: string;
  enValue: string;
  onChange: (lang: 'ar' | 'en', value: string) => void;
  arPlaceholder?: string;
  enPlaceholder?: string;
  arLabel?: string;
  enLabel?: string;
  multiline?: boolean;
}

export function BilingualTextField({
  arValue,
  enValue,
  onChange,
  arPlaceholder = 'أدخل النص العربي',
  enPlaceholder = 'Enter English text',
  arLabel = 'عربي',
  enLabel = 'English',
  multiline = false,
}: BilingualTextFieldProps) {
  const InputComponent = multiline ? 'textarea' : 'input';

  return (
    <div className="bilingual-field">
      {/* Arabic Field (RTL) */}
      <div className="field-wrapper" dir="rtl">
        <label className="field-label rtl-label">
          {arLabel}
        </label>
        <InputComponent
          value={arValue}
          onChange={(e) => onChange('ar', e.target.value)}
          placeholder={arPlaceholder}
          dir="rtl"
          lang="ar"
          className="text-input rtl-input"
          style={{ 
            fontFamily: 'var(--font-arabic)',
            textAlign: 'right',
          }}
        />
      </div>

      {/* English Field (LTR) */}
      <div className="field-wrapper" dir="ltr">
        <label className="field-label ltr-label">
          {enLabel}
        </label>
        <InputComponent
          value={enValue}
          onChange={(e) => onChange('en', e.target.value)}
          placeholder={enPlaceholder}
          dir="ltr"
          lang="en"
          className="text-input ltr-input"
          style={{ 
            fontFamily: 'var(--font-sans)',
            textAlign: 'left',
          }}
        />
      </div>
    </div>
  );
}
```

### Content Block with Drag-and-Drop

```typescript
import { Reorder, useDragControls } from 'framer-motion';
import { GripVertical } from 'lucide-react';

interface ContentBlock {
  id: string;
  type: 'text' | 'image' | 'video';
  content: {
    ar: string;
    en: string;
  };
}

interface ContentBlockEditorProps {
  blocks: ContentBlock[];
  onChange: (blocks: ContentBlock[]) => void;
}

export function ContentBlockEditor({ 
  blocks, 
  onChange 
}: ContentBlockEditorProps) {
  return (
    <Reorder.Group
      axis="y"
      values={blocks}
      onReorder={onChange}
      className="content-blocks"
    >
      {blocks.map((block) => (
        <ContentBlockItem
          key={block.id}
          block={block}
          onUpdate={(updated) => {
            onChange(
              blocks.map((b) => (b.id === block.id ? updated : b))
            );
          }}
          onDelete={() => {
            onChange(blocks.filter((b) => b.id !== block.id));
          }}
        />
      ))}
    </Reorder.Group>
  );
}

function ContentBlockItem({
  block,
  onUpdate,
  onDelete,
}: {
  block: ContentBlock;
  onUpdate: (block: ContentBlock) => void;
  onDelete: () => void;
}) {
  const dragControls = useDragControls();

  return (
    <Reorder.Item
      value={block}
      dragListener={false}
      dragControls={dragControls}
      className="content-block-item"
    >
      {/* Drag Handle */}
      <button
        className="drag-handle"
        onPointerDown={(e) => dragControls.start(e)}
      >
        <GripVertical size={20} />
      </button>

      {/* Block Content */}
      <div className="block-content">
        {block.type === 'text' && (
          <BilingualTextField
            arValue={block.content.ar}
            enValue={block.content.en}
            onChange={(lang, value) => {
              onUpdate({
                ...block,
                content: { ...block.content, [lang]: value },
              });
            }}
            multiline
          />
        )}
        
        {/* Other block types... */}
      </div>

      {/* Delete Button */}
      <button className="delete-button" onClick={onDelete}>
        Remove
      </button>
    </Reorder.Item>
  );
}
```

### CSS for Bidirectional Layout

```css
/**
 * Using CSS Logical Properties for direction-agnostic layout
 * These adapt automatically to dir="ltr" or dir="rtl"
 */

.bilingual-field {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1rem;
}

.field-wrapper {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

/* Logical properties work in both directions */
.field-label {
  padding-inline-start: 0.5rem;  /* Left in LTR, Right in RTL */
}

.text-input {
  padding: 0.75rem 1rem;
  border: 1px solid var(--border-color);
  border-radius: 0.5rem;
}

/* RTL-specific styling */
.rtl-input {
  font-family: 'Noto Sans Arabic', 'Geist', sans-serif;
  line-height: 1.8;  /* Arabic text needs more line height */
}

/* Drag handle styling (always LTR positioning) */
.drag-handle {
  position: absolute;
  inset-inline-start: -2rem;  /* Logical property */
  cursor: grab;
}

.drag-handle:active {
  cursor: grabbing;
}

/* Content block animation */
.content-block-item {
  position: relative;
  padding: 1rem;
  background: var(--surface-color);
  border-radius: 0.5rem;
  
  /* Smooth reorder animation */
  transition: box-shadow 0.2s ease;
}

.content-block-item[data-dragging="true"] {
  box-shadow: 0 8px 24px rgba(0, 0, 0, 0.2);
  z-index: 100;
}
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Key Techniques Used

#### 1. Explicit Direction Attributes

```typescript
// Always set dir explicitly
<div dir="rtl">
  <input dir="rtl" lang="ar" />
</div>

// Never rely on auto-detection for form fields
// ❌ <input dir="auto" />  // Unreliable
```

*Why it matters:* Auto-detection fails on empty fields or fields starting with numbers.

#### 2. CSS Logical Properties

```css
/* Physical properties (direction-specific) */
padding-left: 1rem;    /* Always left */
margin-right: 1rem;    /* Always right */

/* Logical properties (direction-aware) */
padding-inline-start: 1rem;  /* Left in LTR, Right in RTL */
margin-inline-end: 1rem;     /* Right in LTR, Left in RTL */
```

*Why it matters:* One CSS rule that works for both directions.

#### 3. Framer Motion Drag Controls

```typescript
const dragControls = useDragControls();

// Separate drag listener from the item
<Reorder.Item
  dragListener={false}           // Don't drag on item click
  dragControls={dragControls}    // Use explicit controls
>
  <button 
    onPointerDown={(e) => dragControls.start(e)}  // Only drag from handle
  >
    ⋮⋮
  </button>
</Reorder.Item>
```

*Why it matters:* Users can interact with form fields without accidentally dragging.


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Testing Approach

```typescript
describe('BilingualTextField', () => {
  it('should apply RTL direction to Arabic field', () => {
    const { container } = render(
      <BilingualTextField 
        arValue="" 
        enValue="" 
        onChange={jest.fn()} 
      />
    );

    const arField = container.querySelector('[lang="ar"]');
    expect(arField).toHaveAttribute('dir', 'rtl');
  });

  it('should update correct language on change', () => {
    const onChange = jest.fn();
    const { getByPlaceholderText } = render(
      <BilingualTextField 
        arValue="" 
        enValue="" 
        onChange={onChange} 
      />
    );

    fireEvent.change(getByPlaceholderText(/العربي/), {
      target: { value: 'مرحبا' }
    });

    expect(onChange).toHaveBeenCalledWith('ar', 'مرحبا');
  });
});

describe('ContentBlockEditor', () => {
  it('should reorder blocks on drag', async () => {
    const blocks = [
      { id: '1', type: 'text', content: { ar: '', en: '' } },
      { id: '2', type: 'text', content: { ar: '', en: '' } },
    ];
    const onChange = jest.fn();

    render(<ContentBlockEditor blocks={blocks} onChange={onChange} />);

    // Simulate drag from first to second position
    // (Using testing-library's drag simulation)
    
    expect(onChange).toHaveBeenCalledWith([
      expect.objectContaining({ id: '2' }),
      expect.objectContaining({ id: '1' }),
    ]);
  });
});
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Performance Considerations

**Optimization for Complex Content:**

1. **Virtualization for Long Lists**
```typescript
// For 100+ content blocks
import { Virtuoso } from 'react-virtuoso';

<Virtuoso
  data={blocks}
  itemContent={(index, block) => (
    <ContentBlockItem block={block} />
  )}
/>
```

2. **Debounced Text Updates**
```typescript
const debouncedUpdate = useDebouncedCallback(onUpdate, 300);
```

3. **Memoized Block Components**
```typescript
const MemoizedBlockItem = React.memo(ContentBlockItem, (prev, next) => {
  return prev.block.id === next.block.id && 
         prev.block.content === next.block.content;
});
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Lessons Learned

### What Worked Well
- **Explicit direction** is more reliable than auto-detection
- **CSS Logical Properties** dramatically simplify bidirectional styling
- **Framer Motion Reorder** makes drag-and-drop accessible
- **Side-by-side editing** prevents translation gaps

### What I'd Do Differently
- Add **visual connection** between AR/EN versions
- Implement **translation suggestions** (API integration)
- Add **preview mode** showing final bilingual layout
- Support **mixed-direction** blocks (Hebrew + Arabic + English)

### Key Takeaways
1. Bidirectional layouts need explicit direction, not auto-detection
2. CSS Logical Properties should be the default for new projects
3. Drag-and-drop needs explicit handles when editing forms
4. Test with actual RTL text, not just LTR placeholder text


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Common Pitfalls

### 1. Relying on dir="auto"

```html
<!-- ❌ BAD: Auto doesn't work on empty or numeric-starting strings -->
<input dir="auto" value="" />
<input dir="auto" value="123 test" />  <!-- Detected as LTR! -->

<!-- ✅ GOOD: Explicit direction -->
<input dir="rtl" lang="ar" value="" />
```

### 2. Using Physical CSS Properties

```css
/* ❌ BAD: Breaks in RTL */
.button { margin-left: 10px; }

/* ✅ GOOD: Works in both directions */
.button { margin-inline-start: 10px; }
```

### 3. Drag Affecting Text Selection

```typescript
// ❌ BAD: Dragging interferes with text selection
<Reorder.Item>
  <textarea /> {/* Can't select text, dragging starts */}
</Reorder.Item>

// ✅ GOOD: Drag only on explicit handle
<Reorder.Item dragListener={false}>
  <GripHandle onPointerDown={(e) => controls.start(e)} />
  <textarea /> {/* Text selection works */}
</Reorder.Item>
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Related Patterns

- [Schema-Driven Forms](./schema-driven-forms.md) - How bilingual fields are rendered
- [Pending Media Context](./pending-media-context.md) - Image handling in blocks
- [Canvas Background](./canvas-background.md) - Visual polish companion


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Further Reading

- [CSS Logical Properties - MDN](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Logical_Properties)
- [Bidirectional Text - W3C](https://www.w3.org/International/articles/inline-bidi-markup/)
- [Framer Motion Reorder](https://www.framer.com/motion/reorder/)
- [RTL Styling Guide](https://rtlstyling.com/posts/rtl-styling)


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


*Sample complexity: High | Education value: High | Interview relevance: Medium*
