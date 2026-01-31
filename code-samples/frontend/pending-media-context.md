# Pending Media Context Pattern

## Context

**Feature:** Staged Image Uploads  
**Complexity:** Medium  
**Technologies Used:** React Context, Blob URLs, FormData, ObjectURL Management

**The Challenge:**
Implement a staging system where users can upload multiple images, preview them instantly, and only commit them to the server when saving the entire form.


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## The Problem

Traditional file upload approaches have UX issues:

```
Immediate Upload Approach:
1. User adds image    → Uploaded immediately
2. User removes image → Delete API call
3. User cancels form  → Orphaned files on server!
4. User adds 5 images → 5 separate API calls
```

**Requirements:**
- Instant preview without server round-trip
- Batch upload on form save
- Track images for deletion
- Clean up resources on component unmount

**Constraints:**
- Images can be large (10MB+)
- Multiple images per form
- Need to handle both new uploads and deletions
- ObjectURLs must be revoked to prevent memory leaks


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## The Approach

**"Stage Locally, Commit Globally"** - Images are staged in browser memory until the user saves the form.

### Why This Pattern?

```
With Pending Media Context:
1. User adds image    → Blob URL created (instant)
2. User adds 5 images → All stored in context
3. User saves form    → Single batch upload
4. User cancels form  → Nothing uploaded!
```

### Alternatives Considered

1. **Immediate Upload Per Image**
   - Pros: Simple, image already on server
   - Cons: Orphaned files on cancel, many API calls

2. **Base64 Encoding in State**
   - Pros: Can serialize to JSON
   - Cons: Slow, large memory footprint

3. **Blob URL with Context** ✅
   - Pros: Instant preview, batch upload, clean cancel
   - Trade-offs: Must manage ObjectURL lifecycle


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Implementation (Simplified)

### Context Definition

```typescript
/**
 * DISCLAIMER: This is a simplified version for demonstration purposes.
 * The actual implementation includes:
 * - Integration with form state
 * - Image compression before staging
 * - Duplicate detection
 */

import React, { 
  createContext, 
  useCallback, 
  useContext, 
  useState 
} from 'react';

// A pending image ready for upload
interface PendingImage {
  id: string;           // Unique identifier
  file: File;           // Original File object
  previewUrl: string;   // Blob URL for preview
  targetPath: string;   // Where it will be saved
  fieldPath: string;    // Which form field it belongs to
}

interface PendingMediaContextType {
  // Pending images to upload
  pendingImages: PendingImage[];
  addPendingImage: (fieldPath: string, file: File, targetPath: string) => string;
  removePendingImage: (id: string) => void;
  clearPending: () => void;

  // Paths of images to delete
  deletedPaths: string[];
  markForDeletion: (path: string) => void;
  unmarkDeletion: (path: string) => void;
  clearDeletions: () => void;
}

const PendingMediaContext = createContext<PendingMediaContextType | null>(null);

export function PendingMediaProvider({ 
  children 
}: { 
  children: React.ReactNode 
}) {
  const [pendingImages, setPendingImages] = useState<PendingImage[]>([]);
  const [deletedPaths, setDeletedPaths] = useState<string[]>([]);

  // ═══════════════════════════════════════════════════════════════
  // ADD: Stage a new image for upload
  // ═══════════════════════════════════════════════════════════════
  const addPendingImage = useCallback(
    (fieldPath: string, file: File, targetPath: string): string => {
      // Create unique ID for this pending upload
      const id = `pending-${Date.now()}-${Math.random().toString(36).slice(2)}`;
      
      // Create Blob URL for instant preview
      const previewUrl = URL.createObjectURL(file);

      const newImage: PendingImage = {
        id,
        file,
        previewUrl,
        targetPath,
        fieldPath,
      };

      setPendingImages((prev) => [...prev, newImage]);

      // Return the preview URL for the form to use
      return previewUrl;
    },
    []
  );

  // ═══════════════════════════════════════════════════════════════
  // REMOVE: Cancel a pending upload
  // ═══════════════════════════════════════════════════════════════
  const removePendingImage = useCallback((id: string) => {
    setPendingImages((prev) => {
      const image = prev.find((img) => img.id === id);
      if (image) {
        // CRITICAL: Revoke the Blob URL to free memory
        URL.revokeObjectURL(image.previewUrl);
      }
      return prev.filter((img) => img.id !== id);
    });
  }, []);

  // ═══════════════════════════════════════════════════════════════
  // CLEAR: Reset all pending on save/cancel
  // ═══════════════════════════════════════════════════════════════
  const clearPending = useCallback(() => {
    // Revoke ALL Blob URLs before clearing
    setPendingImages((prev) => {
      prev.forEach((img) => URL.revokeObjectURL(img.previewUrl));
      return [];
    });
  }, []);

  // ═══════════════════════════════════════════════════════════════
  // DELETION: Mark existing images for removal
  // ═══════════════════════════════════════════════════════════════
  const markForDeletion = useCallback((path: string) => {
    setDeletedPaths((prev) => {
      if (prev.includes(path)) return prev;
      return [...prev, path];
    });
  }, []);

  const unmarkDeletion = useCallback((path: string) => {
    setDeletedPaths((prev) => prev.filter((p) => p !== path));
  }, []);

  const clearDeletions = useCallback(() => {
    setDeletedPaths([]);
  }, []);

  return (
    <PendingMediaContext.Provider
      value={{
        pendingImages,
        addPendingImage,
        removePendingImage,
        clearPending,
        deletedPaths,
        markForDeletion,
        unmarkDeletion,
        clearDeletions,
      }}
    >
      {children}
    </PendingMediaContext.Provider>
  );
}

// Hook for consuming the context
export function usePendingMedia() {
  const context = useContext(PendingMediaContext);
  if (!context) {
    throw new Error('usePendingMedia must be used within PendingMediaProvider');
  }
  return context;
}
```

### Image Input Component

```typescript
interface ImageInputProps {
  value: string;           // Current image path
  onChange: (path: string) => void;
  uploadPath: string;      // Target directory
  fieldPath: string;       // Form field identifier
}

export function ImageInput({
  value,
  onChange,
  uploadPath,
  fieldPath,
}: ImageInputProps) {
  const { 
    addPendingImage, 
    markForDeletion, 
    pendingImages 
  } = usePendingMedia();
  
  const inputRef = useRef<HTMLInputElement>(null);

  // Check if current value is a pending upload
  const isPending = value?.startsWith('blob:');
  
  // Find pending image by preview URL
  const pendingImage = pendingImages.find(
    (img) => img.previewUrl === value
  );

  const handleFileSelect = (e: React.ChangeEvent<HTMLInputElement>) => {
    const file = e.target.files?.[0];
    if (!file) return;

    // Validate file
    if (!file.type.startsWith('image/')) {
      alert('Please select an image file');
      return;
    }

    if (file.size > 10 * 1024 * 1024) { // 10MB limit
      alert('Image must be smaller than 10MB');
      return;
    }

    // Generate target path
    const extension = file.name.split('.').pop() || 'jpg';
    const filename = `${Date.now()}.${extension}`;
    const targetPath = `${uploadPath}/${filename}`;

    // Stage the image and get preview URL
    const previewUrl = addPendingImage(fieldPath, file, targetPath);

    // Update form state with preview URL
    // (Will be replaced with real path after save)
    onChange(previewUrl);
  };

  const handleRemove = () => {
    if (isPending) {
      // Remove from pending uploads
      if (pendingImage) {
        removePendingImage(pendingImage.id);
      }
    } else if (value) {
      // Mark existing image for deletion
      markForDeletion(value);
    }
    onChange('');
  };

  return (
    <div className="image-input">
      {value ? (
        <div className="image-preview">
          <img 
            src={value} 
            alt="Preview" 
            className="preview-image" 
          />
          {isPending && (
            <span className="pending-badge">Pending Upload</span>
          )}
          <button onClick={handleRemove} className="remove-button">
            Remove
          </button>
        </div>
      ) : (
        <button 
          onClick={() => inputRef.current?.click()}
          className="upload-button"
        >
          Upload Image
        </button>
      )}
      
      <input
        ref={inputRef}
        type="file"
        accept="image/*"
        onChange={handleFileSelect}
        className="hidden"
      />
    </div>
  );
}
```

### Form Save Handler

```typescript
async function handleSave() {
  const { pendingImages, deletedPaths, clearPending, clearDeletions } = 
    usePendingMedia();

  // Build FormData with all pending images
  const formData = new FormData();
  
  // Add JSON data
  formData.append('data', JSON.stringify(formState));

  // Add all pending images
  pendingImages.forEach((img) => {
    formData.append('images', img.file);
    formData.append('imagePaths', img.targetPath);
  });

  // Add deletion list
  formData.append('deletePaths', JSON.stringify(deletedPaths));

  // Single API call for everything
  const response = await fetch('/api/portfolio/batch-save', {
    method: 'POST',
    body: formData,
  });

  if (response.ok) {
    // Clear pending state on success
    clearPending();
    clearDeletions();
    
    // Update form state with real paths
    const { savedPaths } = await response.json();
    // ... update form with savedPaths
  }
}
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Key Techniques Used

#### 1. Blob URL for Instant Preview

```typescript
// Create a URL that points to in-memory file data
const previewUrl = URL.createObjectURL(file);

// Can be used directly in <img src={previewUrl} />
// No network request needed!
```

*Why it matters:* Users see images instantly without waiting for upload.

#### 2. Memory Management with URL.revokeObjectURL

```typescript
// ⚠️ Blob URLs consume memory until revoked
const url = URL.createObjectURL(file);  // Memory allocated

// When done with the URL:
URL.revokeObjectURL(url);  // Memory freed

// Common bug: forgetting to revoke on component unmount
```

*Why it matters:* Prevents memory leaks in long-running sessions.

#### 3. Dual-Track State (Pending vs Deleted)

```typescript
// Track NEW images (to be uploaded)
pendingImages: PendingImage[]

// Track EXISTING images (to be deleted)
deletedPaths: string[]

// On save: upload pending, delete marked
const operations = [
  ...pendingImages.map(img => ({ path: img.targetPath, content: img.file })),
  ...deletedPaths.map(path => ({ path, content: null })),  // null = delete
];
```

*Why it matters:* Clean abstraction for handling both operations.


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Testing Approach

```typescript
describe('PendingMediaContext', () => {
  it('should create blob URL when image added', () => {
    const { result } = renderHook(() => usePendingMedia(), {
      wrapper: PendingMediaProvider,
    });

    const file = new File(['test'], 'test.jpg', { type: 'image/jpeg' });
    
    act(() => {
      result.current.addPendingImage('hero', file, 'images/hero.jpg');
    });

    expect(result.current.pendingImages).toHaveLength(1);
    expect(result.current.pendingImages[0].previewUrl).toMatch(/^blob:/);
  });

  it('should revoke blob URL when image removed', () => {
    const revokeObjectURL = jest.spyOn(URL, 'revokeObjectURL');
    const { result } = renderHook(() => usePendingMedia(), {
      wrapper: PendingMediaProvider,
    });

    const file = new File(['test'], 'test.jpg', { type: 'image/jpeg' });
    
    let previewUrl: string;
    act(() => {
      previewUrl = result.current.addPendingImage('hero', file, 'images/hero.jpg');
    });

    act(() => {
      const id = result.current.pendingImages[0].id;
      result.current.removePendingImage(id);
    });

    expect(revokeObjectURL).toHaveBeenCalledWith(previewUrl);
  });

  it('should track deleted paths separately', () => {
    const { result } = renderHook(() => usePendingMedia(), {
      wrapper: PendingMediaProvider,
    });

    act(() => {
      result.current.markForDeletion('images/old.jpg');
      result.current.markForDeletion('images/older.jpg');
    });

    expect(result.current.deletedPaths).toEqual([
      'images/old.jpg',
      'images/older.jpg',
    ]);
  });
});
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Performance Considerations

**Memory Usage:**
| Scenario | Traditional (Base64) | Blob URL |
|----------|---------------------|----------|
| 5MB image | ~6.7MB in state | 5MB in browser blob store |
| 10 images | ~67MB JSON state | Separate from React state |
| Serialization | Slow (large JSON) | Fast (just URLs) |

**Best Practices:**
1. Revoke Blob URLs when no longer needed
2. Use `useCallback` for context methods
3. Batch operations in single save call
4. Clear pending on unmount


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Lessons Learned

### What Worked Well
- **Instant preview** dramatically improves perceived performance
- **Batch save** reduces API complexity
- **Cancel = free** - no cleanup needed on abandoned edits
- **Separation of pending vs deleted** is clean mental model

### What I'd Do Differently
- Add **image compression** before staging
- Implement **progress indicators** for large files
- Add **drag-and-drop** support at context level
- Consider **IndexedDB** for persistence across page refreshes

### Key Takeaways
1. Blob URLs are underutilized for local file handling
2. Memory management (revokeObjectURL) is critical
3. Context is ideal for cross-cutting concerns like media staging
4. Separating "to add" from "to delete" simplifies logic


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Common Pitfalls

### 1. Memory Leaks from Unreleased Blob URLs

```typescript
// ❌ BAD: Never revokes URL
const url = URL.createObjectURL(file);
setImages(prev => [...prev, { url }]);
// Memory leak when images change!

// ✅ GOOD: Revoke on removal
const removeImage = (id) => {
  setImages(prev => {
    const img = prev.find(i => i.id === id);
    URL.revokeObjectURL(img.url);  // Free memory
    return prev.filter(i => i.id !== id);
  });
};
```

### 2. Confusing Blob URLs with Real Paths

```typescript
// ❌ BAD: Saving blob URL to database
save({ heroImage: 'blob:https://...' }); // Won't work on reload!

// ✅ GOOD: Replace blob URLs with real paths after upload
const savedPaths = await uploadImages(pendingImages);
save({ heroImage: savedPaths['hero'] }); // Real server path
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Related Patterns

- [Schema-Driven Forms](./schema-driven-forms.md) - Where ImageInput is used
- [GitHub Batch Commits](../backend/github-batch-commits.md) - How images are saved
- [UI Context System](../utilities/ui-context-system.md) - Similar context pattern


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Further Reading

- [URL.createObjectURL() - MDN](https://developer.mozilla.org/en-US/docs/Web/API/URL/createObjectURL)
- [File API - MDN](https://developer.mozilla.org/en-US/docs/Web/API/File)
- [React Context Best Practices](https://react.dev/learn/passing-data-deeply-with-context)


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


*Sample complexity: Medium | Education value: High | Interview relevance: Medium*
