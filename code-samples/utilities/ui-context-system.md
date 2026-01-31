# Global UI Context System

## Context

**Feature:** Centralized UI State Management  
**Complexity:** Medium  
**Technologies Used:** React Context, AnimatePresence, Promise-based Dialogs

**The Challenge:**
Provide global access to UI primitives (toasts, modals, confirmations) from anywhere in the app, including non-component code, with smooth animations.


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## The Problem

UI feedback is needed everywhere, but components are deeply nested:

```
App
  └── Layout
       └── Page
            └── Form
                 └── Button (needs to show toast!)
```

**Requirements:**
- Show toast notifications from any component
- Display confirmation dialogs with async responses
- Manage modal visibility globally
- Animate entrances/exits smoothly

**Constraints:**
- Can't use `window.alert()` (ugly, blocking)
- Must work with React Server Components (RSC)
- Should support stacking multiple toasts
- Must be accessible (ARIA, focus management)


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## The Approach

**Context + Portals + AnimatePresence** - Centralized state with decentralized rendering.

### Why This Pattern?

```typescript
// From ANY component:
const { showToast, showConfirm } = useUI();

// Toast example
showToast('Changes saved!', 'success');

// Async confirm example
const confirmed = await showConfirm('Delete this item?');
if (confirmed) {
  // proceed
}
```

### Alternatives Considered

1. **Event-Based System (EventEmitter)**
   - Pros: Works from non-React code
   - Cons: Memory leak risk, harder to type

2. **Third-Party Library (react-toastify, etc.)**
   - Pros: Feature-rich, well-tested
   - Cons: Additional dependency, styling conflicts

3. **Custom Context System** ✅
   - Pros: Full control, matches design system, small bundle
   - Trade-offs: More code to write and maintain


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Implementation (Simplified)

### Context Definition

```typescript
/**
 * DISCLAIMER: This is a simplified version for demonstration purposes.
 * The actual implementation includes:
 * - Accessibility announcements
 * - Focus management for modals
 * - Toast queuing and stacking
 */

'use client';

import React, { 
  createContext, 
  useContext, 
  useState, 
  useCallback 
} from 'react';
import { AnimatePresence, motion } from 'framer-motion';

// ═══════════════════════════════════════════════════════════════════
// TYPES
// ═══════════════════════════════════════════════════════════════════

type ToastType = 'success' | 'error' | 'info' | 'warning';

interface Toast {
  id: string;
  message: string;
  type: ToastType;
}

interface ConfirmState {
  message: string;
  resolve: (value: boolean) => void;
}

interface UIContextType {
  // Toast management
  showToast: (message: string, type?: ToastType) => void;
  
  // Confirmation dialogs
  showConfirm: (message: string) => Promise<boolean>;
  
  // Generic modal control
  openModal: (content: React.ReactNode) => void;
  closeModal: () => void;
}

const UIContext = createContext<UIContextType | null>(null);

// ═══════════════════════════════════════════════════════════════════
// PROVIDER
// ═══════════════════════════════════════════════════════════════════

export function UIProvider({ children }: { children: React.ReactNode }) {
  const [toasts, setToasts] = useState<Toast[]>([]);
  const [confirm, setConfirm] = useState<ConfirmState | null>(null);
  const [modalContent, setModalContent] = useState<React.ReactNode | null>(null);

  // ─────────────────────────────────────────────────────────────────
  // TOAST: Show temporary notification
  // ─────────────────────────────────────────────────────────────────
  
  const showToast = useCallback((message: string, type: ToastType = 'info') => {
    const id = `toast-${Date.now()}-${Math.random().toString(36).slice(2)}`;
    
    setToasts((prev) => [...prev, { id, message, type }]);
    
    // Auto-dismiss after 4 seconds
    setTimeout(() => {
      setToasts((prev) => prev.filter((t) => t.id !== id));
    }, 4000);
  }, []);

  // ─────────────────────────────────────────────────────────────────
  // CONFIRM: Promise-based confirmation dialog
  // ─────────────────────────────────────────────────────────────────
  
  const showConfirm = useCallback((message: string): Promise<boolean> => {
    return new Promise((resolve) => {
      setConfirm({ message, resolve });
    });
  }, []);

  const handleConfirmResponse = (response: boolean) => {
    confirm?.resolve(response);
    setConfirm(null);
  };

  // ─────────────────────────────────────────────────────────────────
  // MODAL: Generic modal content
  // ─────────────────────────────────────────────────────────────────
  
  const openModal = useCallback((content: React.ReactNode) => {
    setModalContent(content);
  }, []);

  const closeModal = useCallback(() => {
    setModalContent(null);
  }, []);

  return (
    <UIContext.Provider 
      value={{ showToast, showConfirm, openModal, closeModal }}
    >
      {children}

      {/* Toast Container */}
      <div 
        className="fixed top-4 right-4 z-50 flex flex-col gap-2"
        role="region"
        aria-label="Notifications"
      >
        <AnimatePresence mode="popLayout">
          {toasts.map((toast) => (
            <motion.div
              key={toast.id}
              initial={{ opacity: 0, y: -20, scale: 0.9 }}
              animate={{ opacity: 1, y: 0, scale: 1 }}
              exit={{ opacity: 0, x: 100, scale: 0.9 }}
              className={`toast toast-${toast.type}`}
              role="alert"
            >
              {toast.message}
            </motion.div>
          ))}
        </AnimatePresence>
      </div>

      {/* Confirm Dialog */}
      <AnimatePresence>
        {confirm && (
          <motion.div
            initial={{ opacity: 0 }}
            animate={{ opacity: 1 }}
            exit={{ opacity: 0 }}
            className="fixed inset-0 z-50 flex items-center justify-center bg-black/50"
            onClick={() => handleConfirmResponse(false)}
          >
            <motion.div
              initial={{ scale: 0.9, opacity: 0 }}
              animate={{ scale: 1, opacity: 1 }}
              exit={{ scale: 0.9, opacity: 0 }}
              className="bg-white dark:bg-gray-800 p-6 rounded-lg shadow-xl max-w-md"
              onClick={(e) => e.stopPropagation()}
              role="alertdialog"
              aria-modal="true"
            >
              <p className="text-lg mb-4">{confirm.message}</p>
              <div className="flex gap-3 justify-end">
                <button
                  onClick={() => handleConfirmResponse(false)}
                  className="px-4 py-2 text-gray-600 hover:bg-gray-100 rounded"
                >
                  Cancel
                </button>
                <button
                  onClick={() => handleConfirmResponse(true)}
                  className="px-4 py-2 bg-red-600 text-white rounded hover:bg-red-700"
                  autoFocus
                >
                  Confirm
                </button>
              </div>
            </motion.div>
          </motion.div>
        )}
      </AnimatePresence>

      {/* Generic Modal */}
      <AnimatePresence>
        {modalContent && (
          <motion.div
            initial={{ opacity: 0 }}
            animate={{ opacity: 1 }}
            exit={{ opacity: 0 }}
            className="fixed inset-0 z-50 flex items-center justify-center bg-black/50"
            onClick={closeModal}
          >
            <motion.div
              initial={{ scale: 0.9, opacity: 0 }}
              animate={{ scale: 1, opacity: 1 }}
              exit={{ scale: 0.9, opacity: 0 }}
              onClick={(e) => e.stopPropagation()}
            >
              {modalContent}
            </motion.div>
          </motion.div>
        )}
      </AnimatePresence>
    </UIContext.Provider>
  );
}

// ═══════════════════════════════════════════════════════════════════
// HOOK
// ═══════════════════════════════════════════════════════════════════

export function useUI() {
  const context = useContext(UIContext);
  if (!context) {
    throw new Error('useUI must be used within a UIProvider');
  }
  return context;
}
```

### Usage Examples

```typescript
// ─────────────────────────────────────────────────────────────────
// Example: Toast Notifications
// ─────────────────────────────────────────────────────────────────

function SaveButton() {
  const { showToast } = useUI();

  const handleSave = async () => {
    try {
      await saveData();
      showToast('Changes saved successfully!', 'success');
    } catch (error) {
      showToast('Failed to save changes', 'error');
    }
  };

  return <button onClick={handleSave}>Save</button>;
}

// ─────────────────────────────────────────────────────────────────
// Example: Confirmation Before Dangerous Action
// ─────────────────────────────────────────────────────────────────

function DeleteButton({ itemId }: { itemId: string }) {
  const { showConfirm, showToast } = useUI();

  const handleDelete = async () => {
    // Await user's decision
    const confirmed = await showConfirm(
      'Are you sure you want to delete this item? This action cannot be undone.'
    );

    if (confirmed) {
      await deleteItem(itemId);
      showToast('Item deleted', 'info');
    }
    // If not confirmed, nothing happens
  };

  return <button onClick={handleDelete}>Delete</button>;
}

// ─────────────────────────────────────────────────────────────────
// Example: Custom Modal Content
// ─────────────────────────────────────────────────────────────────

function ProfileButton() {
  const { openModal, closeModal } = useUI();

  const handleOpenProfile = () => {
    openModal(
      <div className="bg-white p-6 rounded-lg max-w-md">
        <h2>Edit Profile</h2>
        <form>
          {/* Form fields */}
        </form>
        <button onClick={closeModal}>Close</button>
      </div>
    );
  };

  return <button onClick={handleOpenProfile}>Edit Profile</button>;
}
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Key Techniques Used

#### 1. Promise-Based Confirm Dialog

```typescript
// The magic: resolve is stored in state
const showConfirm = (message: string) => {
  return new Promise((resolve) => {
    setConfirm({ message, resolve }); // Store resolve function
  });
};

// When user clicks, call the stored resolve
const handleConfirmResponse = (response: boolean) => {
  confirm?.resolve(response);  // Resolves the original Promise
  setConfirm(null);
};

// Usage - await the Promise!
const confirmed = await showConfirm('Delete?');
```

*Why it matters:* Imperative-style async confirmations in declarative React.

#### 2. AnimatePresence for Exit Animations

```typescript
<AnimatePresence mode="popLayout">
  {toasts.map((toast) => (
    <motion.div
      key={toast.id}
      exit={{ opacity: 0, x: 100 }}  // Animate out when removed
    >
      {toast.message}
    </motion.div>
  ))}
</AnimatePresence>
```

*Why it matters:* Elements animate out before being removed from DOM.

#### 3. Click-Through Backdrop

```typescript
// Backdrop closes modal on click
<div onClick={closeModal} className="fixed inset-0">
  
  {/* But modal content stops propagation */}
  <div onClick={(e) => e.stopPropagation()}>
    {/* Modal stays open when clicking inside */}
  </div>
  
</div>
```

*Why it matters:* Natural UX - click outside to dismiss.


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Testing Approach

```typescript
describe('UIContext', () => {
  describe('showToast', () => {
    it('should display toast message', () => {
      const { result } = renderHook(() => useUI(), {
        wrapper: UIProvider,
      });

      act(() => {
        result.current.showToast('Test message', 'success');
      });

      // Toast should be visible (test actual DOM)
    });

    it('should auto-dismiss after 4 seconds', async () => {
      jest.useFakeTimers();
      
      const { result } = renderHook(() => useUI(), {
        wrapper: UIProvider,
      });

      act(() => {
        result.current.showToast('Test message');
      });

      // Fast-forward time
      act(() => {
        jest.advanceTimersByTime(4000);
      });

      // Toast should be gone
    });
  });

  describe('showConfirm', () => {
    it('should resolve true when confirmed', async () => {
      const { result } = renderHook(() => useUI(), {
        wrapper: UIProvider,
      });

      let confirmPromise: Promise<boolean>;
      act(() => {
        confirmPromise = result.current.showConfirm('Confirm?');
      });

      // Simulate clicking confirm button
      const confirmButton = screen.getByText('Confirm');
      fireEvent.click(confirmButton);

      await expect(confirmPromise!).resolves.toBe(true);
    });

    it('should resolve false when cancelled', async () => {
      const { result } = renderHook(() => useUI(), {
        wrapper: UIProvider,
      });

      let confirmPromise: Promise<boolean>;
      act(() => {
        confirmPromise = result.current.showConfirm('Confirm?');
      });

      const cancelButton = screen.getByText('Cancel');
      fireEvent.click(cancelButton);

      await expect(confirmPromise!).resolves.toBe(false);
    });
  });
});
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Performance Considerations

**Memory Management:**
- Toasts auto-dismiss (no manual cleanup needed)
- Confirm dialogs clear state on response
- Modals unmount content when closed

**Animation Performance:**
- Using `transform` and `opacity` (GPU-accelerated)
- `popLayout` mode prevents layout shift
- AnimatePresence only affects exiting elements

**Bundle Size:**
- Framer Motion: ~40KB gzipped
- This context: ~2KB
- Total: Reasonable for the functionality


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Lessons Learned

### What Worked Well
- **Promise-based confirm** is intuitive for developers
- **AnimatePresence** makes exits feel polished
- **Centralized rendering** keeps z-index manageable
- **Context pattern** avoids prop drilling

### What I'd Do Differently
- Add **toast queue** with max visible limit
- Implement **focus trap** for modals (accessibility)
- Add **keyboard support** (Escape to close)
- Consider **portal** for SSR compatibility

### Key Takeaways
1. Storing resolve functions enables imperative patterns in React
2. AnimatePresence is essential for exit animations
3. Click propagation control is crucial for modal UX
4. ARIA attributes improve accessibility significantly


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Common Pitfalls

### 1. Forgetting key for AnimatePresence

```typescript
// ❌ BAD: No key, AnimatePresence can't track items
<AnimatePresence>
  {toasts.map((toast) => (
    <motion.div>{toast.message}</motion.div>  // No key!
  ))}
</AnimatePresence>

// ✅ GOOD: Unique key enables proper enter/exit tracking
<AnimatePresence>
  {toasts.map((toast) => (
    <motion.div key={toast.id}>{toast.message}</motion.div>
  ))}
</AnimatePresence>
```

### 2. Not Stopping Propagation on Modal Content

```typescript
// ❌ BAD: Clicking inside modal closes it
<div onClick={closeModal}>
  <div>{content}</div>  // Click here also closes!
</div>

// ✅ GOOD: Stop propagation on content
<div onClick={closeModal}>
  <div onClick={(e) => e.stopPropagation()}>{content}</div>
</div>
```

### 3. Memory Leak with Unresolved Promises

```typescript
// ❌ BAD: If component unmounts before response
showConfirm('Are you sure?');  // Promise never resolves, state stuck

// ✅ GOOD: Handle cleanup
useEffect(() => {
  return () => {
    confirm?.resolve(false);  // Clean up on unmount
  };
}, [confirm]);
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Related Patterns

- [Pending Media Context](../frontend/pending-media-context.md) - Similar context pattern
- [Schema-Driven Forms](../frontend/schema-driven-forms.md) - Uses toasts for feedback


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Further Reading

- [React Context - Official Docs](https://react.dev/learn/passing-data-deeply-with-context)
- [Framer Motion AnimatePresence](https://www.framer.com/motion/animate-presence/)
- [ARIA Modal Dialog](https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Roles/dialog_role)


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


*Sample complexity: Medium | Education value: High | Interview relevance: Medium*
