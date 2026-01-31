# Schema-Driven Form Engine

## Context

**Feature:** Dynamic Form Generation  
**Complexity:** Very High  
**Technologies Used:** React, TypeScript, JSON Schema, Dynamic Component Rendering

**The Challenge:**
Build a form system that automatically generates the correct UI components from a JSON schema, eliminating the need to write new form code when data models change.


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## The Problem

Traditional form development follows a repetitive pattern:

```
For each new field in the database:
1. Update the data model   → 15 minutes
2. Create/update API       → 30 minutes
3. Add form field in React → 30 minutes
4. Add validation          → 15 minutes
5. Test everything         → 30 minutes
                           ─────────────
Total:                     ~2 hours per field
```

**Requirements:**
- Add new form fields without code changes
- Support multiple field types (text, image, arrays, nested objects)
- Handle bilingual content (RTL Arabic + LTR English)
- Preview images before upload
- Enable drag-and-drop reordering

**Constraints:**
- Must be type-safe with TypeScript
- Schema comes from external source (potentially changes)
- Need to support deeply nested structures
- Performance must remain good with complex forms


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## The Approach

**"The Schema IS the Form"** - Instead of writing forms, we write renderers that interpret schemas.

### Why This Pattern?

```
With Schema-Driven Forms:
1. Update schema JSON       → 5 minutes
2. (Everything else is automatic)

Time savings: ~97% per field
```

### Alternatives Considered

1. **Manual Form Development**
   - Pros: Full control, simple to understand
   - Cons: Repetitive, error-prone, slow to iterate

2. **Form Libraries (React Hook Form, Formik)**
   - Pros: Validation, state management built-in
   - Cons: Still need to define fields manually

3. **Schema-Driven Rendering** ✅
   - Pros: Zero code for new fields, single source of truth
   - Trade-offs: More complex initial implementation, need schema discipline


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Implementation (Simplified)

### Schema Types

```typescript
/**
 * DISCLAIMER: This is a simplified version for demonstration purposes.
 * The actual implementation includes:
 * - Validation rules per field
 * - Custom x-format extensions
 * - Conditional field visibility
 * - Array item schemas
 */

// JSON Schema field definition
interface SchemaField {
  type: 'string' | 'number' | 'boolean' | 'object' | 'array';
  title?: string;
  description?: string;
  format?: 'image' | 'date' | 'url' | 'email';
  // Custom extensions
  'x-format'?: 'detailed-content' | 'bilingual-list';
  'x-upload-path'?: string;
  // For objects
  properties?: Record<string, SchemaField>;
  // For arrays
  items?: SchemaField;
  // Validation
  required?: string[];
  minLength?: number;
  maxLength?: number;
}

// Root schema structure
interface FormSchema {
  type: 'object';
  properties: Record<string, SchemaField>;
  required?: string[];
}
```

### Field Renderer Component

```typescript
import React from 'react';

interface FieldRendererProps {
  field: SchemaField;
  value: unknown;
  onChange: (value: unknown) => void;
  fieldPath: string;
}

/**
 * The core of the schema-driven form engine
 * 
 * Examines the field schema and returns the appropriate component.
 * Think of it as a "component factory" driven by data.
 */
export function FieldRenderer({
  field,
  value,
  onChange,
  fieldPath,
}: FieldRendererProps) {
  // ═══════════════════════════════════════════════════════════════
  // DETECTION: Bilingual Object (has ar/en properties)
  // ═══════════════════════════════════════════════════════════════
  if (isBilingualField(field)) {
    const bilingualValue = (value as BilingualValue) || { ar: '', en: '' };
    return (
      <BilingualTextField
        arValue={bilingualValue.ar}
        enValue={bilingualValue.en}
        onChange={(lang, newValue) => {
          onChange({ ...bilingualValue, [lang]: newValue });
        }}
        arTitle={field.properties?.ar?.title || 'عربي'}
        enTitle={field.properties?.en?.title || 'English'}
      />
    );
  }

  // ═══════════════════════════════════════════════════════════════
  // DETECTION: Image Upload Field
  // ═══════════════════════════════════════════════════════════════
  if (field.format === 'image') {
    return (
      <ImageInput
        value={(value as string) || ''}
        onChange={onChange}
        uploadPath={field['x-upload-path']}
        fieldPath={fieldPath}
      />
    );
  }

  // ═══════════════════════════════════════════════════════════════
  // DETECTION: Array of Strings
  // ═══════════════════════════════════════════════════════════════
  if (field.type === 'array' && field.items?.type === 'string') {
    return (
      <StringArrayEditor
        value={(value as string[]) || []}
        onChange={onChange}
      />
    );
  }

  // ═══════════════════════════════════════════════════════════════
  // DETECTION: Array of Complex Objects
  // ═══════════════════════════════════════════════════════════════
  if (field.type === 'array' && field.items?.type === 'object') {
    return (
      <ObjectArrayEditor
        items={(value as object[]) || []}
        itemSchema={field.items}
        onChange={onChange}
      />
    );
  }

  // ═══════════════════════════════════════════════════════════════
  // DETECTION: Boolean Toggle
  // ═══════════════════════════════════════════════════════════════
  if (field.type === 'boolean') {
    return (
      <Toggle
        checked={(value as boolean) || false}
        onChange={onChange}
        label={field.title || fieldPath}
      />
    );
  }

  // ═══════════════════════════════════════════════════════════════
  // DETECTION: Number Input
  // ═══════════════════════════════════════════════════════════════
  if (field.type === 'number') {
    return (
      <NumberInput
        value={(value as number) || 0}
        onChange={onChange}
        min={field.minimum}
        max={field.maximum}
      />
    );
  }

  // ═══════════════════════════════════════════════════════════════
  // DEFAULT: Simple Text Input
  // ═══════════════════════════════════════════════════════════════
  return (
    <TextInput
      value={(value as string) || ''}
      onChange={(e) => onChange(e.target.value)}
      placeholder={field.title}
    />
  );
}

// Helper to detect bilingual fields
function isBilingualField(field: SchemaField): boolean {
  return (
    field.type === 'object' &&
    field.properties?.ar !== undefined &&
    field.properties?.en !== undefined
  );
}
```

### Dynamic Form Container

```typescript
interface DynamicFormProps {
  schema: FormSchema;
  values: Record<string, unknown>;
  onChange: (values: Record<string, unknown>) => void;
}

export function DynamicForm({ schema, values, onChange }: DynamicFormProps) {
  return (
    <div className="space-y-6">
      {Object.entries(schema.properties).map(([fieldKey, fieldSchema]) => (
        <div key={fieldKey} className="form-field">
          {/* Label */}
          <label className="form-label">
            {fieldSchema.title || fieldKey}
            {schema.required?.includes(fieldKey) && (
              <span className="required-indicator">*</span>
            )}
          </label>

          {/* Description */}
          {fieldSchema.description && (
            <p className="form-description">{fieldSchema.description}</p>
          )}

          {/* Dynamic Field */}
          <FieldRenderer
            field={fieldSchema}
            value={values[fieldKey]}
            onChange={(newValue) => {
              onChange({ ...values, [fieldKey]: newValue });
            }}
            fieldPath={fieldKey}
          />
        </div>
      ))}
    </div>
  );
}
```

### Key Techniques Used

#### 1. Field Type Detection Pattern

```typescript
// Order matters - most specific first
if (isBilingualField(field)) return <BilingualTextField />;
if (field.format === 'image') return <ImageInput />;
if (field.type === 'array') return <ArrayEditor />;
if (field.type === 'boolean') return <Toggle />;
// ... most generic last
return <TextInput />;
```

*Why it matters:* Allows for specialized handling of complex types before falling back to simple ones.

#### 2. Composition over Configuration

```typescript
// Instead of one mega-component with 50 props:
// ❌ <InputField type="bilingual" lang1="ar" lang2="en" rtl ... />

// We compose specialized components:
// ✅ <BilingualTextField arTitle="عربي" enTitle="English" ... />
```

*Why it matters:* Each component is simpler, more testable, and more reusable.

#### 3. Schema Extensions (x-format)

```typescript
// Standard JSON Schema doesn't have everything we need
// Use x-* prefix for custom extensions
{
  "type": "array",
  "x-format": "detailed-content",  // Our custom format
  "x-upload-path": "public/images" // Our custom extension
}
```

*Why it matters:* Extends JSON Schema without breaking compatibility.


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Example Schema

```json
{
  "type": "object",
  "required": ["title", "slug"],
  "properties": {
    "title": {
      "type": "object",
      "properties": {
        "ar": { "type": "string", "title": "العنوان" },
        "en": { "type": "string", "title": "Title" }
      }
    },
    "image": {
      "type": "string",
      "format": "image",
      "x-upload-path": "public/images/projects"
    },
    "tags": {
      "type": "array",
      "items": { "type": "string" }
    },
    "featured": {
      "type": "boolean",
      "title": "Featured Project"
    },
    "year": {
      "type": "number",
      "minimum": 2000,
      "maximum": 2030
    }
  }
}
```

**Generates:**
- Bilingual text field for `title` (AR/EN side by side)
- Image upload for `image`
- Tag list editor for `tags`
- Toggle switch for `featured`
- Number input for `year`


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Testing Approach

```typescript
describe('FieldRenderer', () => {
  it('should render BilingualTextField for ar/en object', () => {
    const field = {
      type: 'object',
      properties: {
        ar: { type: 'string' },
        en: { type: 'string' }
      }
    };

    const { getByPlaceholderText } = render(
      <FieldRenderer field={field} value={{}} onChange={jest.fn()} />
    );

    expect(getByPlaceholderText('عربي')).toBeInTheDocument();
    expect(getByPlaceholderText('English')).toBeInTheDocument();
  });

  it('should render ImageInput for image format', () => {
    const field = {
      type: 'string',
      format: 'image'
    };

    const { getByRole } = render(
      <FieldRenderer field={field} value="" onChange={jest.fn()} />
    );

    expect(getByRole('button', { name: /upload/i })).toBeInTheDocument();
  });

  it('should render StringArrayEditor for string arrays', () => {
    const field = {
      type: 'array',
      items: { type: 'string' }
    };

    const { getByRole } = render(
      <FieldRenderer 
        field={field} 
        value={['item1', 'item2']} 
        onChange={jest.fn()} 
      />
    );

    expect(getByRole('button', { name: /add/i })).toBeInTheDocument();
  });
});
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Performance Considerations

**Optimization Techniques:**

1. **Memoization of Field Components**
```typescript
const MemoizedFieldRenderer = React.memo(FieldRenderer, (prev, next) => {
  return prev.value === next.value && prev.field === next.field;
});
```

2. **Lazy Loading of Complex Editors**
```typescript
const DetailedContentEditor = React.lazy(() => 
  import('./DetailedContentEditor')
);
```

3. **Debounced onChange for Text Inputs**
```typescript
const debouncedOnChange = useDebouncedCallback(onChange, 300);
```

| Metric | Before Optimization | After |
|--------|--------------------|-|
| Re-renders per keystroke | All fields | Only active field |
| Initial load | All editors | Core fields only |
| Form state updates | Immediate | Debounced |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Lessons Learned

### What Worked Well
- **Zero code for new fields** is incredibly liberating
- **Schema as source of truth** keeps frontend and backend in sync
- **TypeScript** catches schema/component mismatches at build time
- **Composition** made testing individual field types easy

### What I'd Do Differently
- Add **schema validation** before rendering (catch errors early)
- Implement **conditional fields** (show field X only if field Y = true)
- Add **custom validation rules** in schema extensions
- Build **schema editor UI** for non-developers

### Key Takeaways
1. Schema-driven UI is worth the upfront investment for evolving data models
2. Detection order matters - specific before generic
3. Custom schema extensions (x-*) are powerful and standard-compatible
4. Composition makes complex forms manageable


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Common Pitfalls

### 1. Detection Order Issues

```typescript
// ❌ BAD: Generic check before specific
if (field.type === 'object') return <ObjectEditor />;
if (isBilingualField(field)) return <BilingualTextField />; // Never reached!

// ✅ GOOD: Specific before generic
if (isBilingualField(field)) return <BilingualTextField />;
if (field.type === 'object') return <ObjectEditor />;
```

### 2. Forgetting Type Coercion

```typescript
// ❌ BAD: Assuming value exists
<TextInput value={value} /> // value might be undefined

// ✅ GOOD: Default to empty
<TextInput value={(value as string) || ''} />
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Related Patterns

- [Bilingual Content Editor](./bilingual-content-editor.md) - RTL/LTR handling
- [Pending Media Context](./pending-media-context.md) - Image staging
- [GitHub Batch Commits](../backend/github-batch-commits.md) - Saving form data


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Further Reading

- [JSON Schema Specification](https://json-schema.org/)
- [Form Design Patterns](https://www.smashingmagazine.com/2021/05/form-design-patterns/)
- [React Dynamic Forms](https://react.dev/learn/sharing-state-between-components)


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


*Sample complexity: Very High | Education value: Very High | Interview relevance: High*
