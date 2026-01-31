# GitHub Batch Commit Engine

## Context

**Feature:** Atomic Multi-File Operations  
**Complexity:** Very High  
**Technologies Used:** Octokit, GitHub App API, Git Trees API, GitHub Data API

**The Challenge:**
Implement a system that can create, update, and delete multiple files in a GitHub repository in a **single atomic commit**, ensuring data consistency and clean commit history.


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## The Problem

Most GitHub integrations use the high-level Contents API, which has a critical limitation:

```
Contents API Limitation:
File 1: Create  → Commit #1
File 2: Update  → Commit #2
File 3: Delete  → Commit #3 ← If this fails, files 1-2 are already committed!
```

**Requirements:**
- Save portfolio data + multiple images in ONE commit
- Support create, update, AND delete operations together
- Maintain atomic transactions (all succeed or none do)
- Keep commit history clean (one commit per logical operation)

**Constraints:**
- Contents API creates one commit per file
- Large files need special handling (blobs)
- Must handle image binary data efficiently
- GitHub API rate limits (5000/hour for apps)


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## The Approach

The key insight: **Use GitHub's low-level Git Data API** instead of the high-level Contents API.

### Why This Pattern?

The Git Data API exposes the same primitives that Git itself uses:
- **Blobs** - File contents
- **Trees** - Directory structure
- **Commits** - Atomic snapshots
- **Refs** - Branch pointers

By working at this level, we can build a commit that includes any number of file operations.

### Alternatives Considered

1. **GitHub Contents API (One Commit Per File)**
   - Pros: Simple API, well-documented
   - Cons: Not atomic, clutters history, slow for multiple files

2. **Git CLI via Server (exec/spawn)**
   - Pros: Full git control
   - Cons: Requires git installation, complex in serverless

3. **Git Data API (Trees API)** ✅
   - Pros: True atomicity, single commit, efficient for batches
   - Trade-offs: More complex, requires understanding Git internals


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Implementation (Simplified)

```typescript
/**
 * DISCLAIMER: This is a simplified version for demonstration purposes.
 * The actual implementation includes:
 * - GitHub App authentication with installation ID caching
 * - Error recovery and retry logic
 * - Detailed logging for debugging
 * - Image processing/optimization
 */

import { Octokit } from '@octokit/rest';

// Operation types for the batch
interface FileOperation {
  path: string;           // File path in repo (e.g., "data/project.json")
  content: string | Buffer | null;  // null = delete
  encoding?: 'utf-8' | 'base64';
}

/**
 * Execute multiple file operations in a single atomic commit
 * 
 * @param octokit - Authenticated Octokit instance
 * @param owner - Repository owner
 * @param repo - Repository name
 * @param branch - Target branch
 * @param message - Commit message
 * @param operations - Array of file operations
 * @returns Created commit SHA
 */
async function batchCommit(
  octokit: Octokit,
  owner: string,
  repo: string,
  branch: string,
  message: string,
  operations: FileOperation[]
): Promise<string> {
  console.log(`[GitHub] Starting batch commit with ${operations.length} operations`);

  // ═══════════════════════════════════════════════════════════════════
  // STEP 1: Get the current commit SHA (the branch's HEAD)
  // ═══════════════════════════════════════════════════════════════════
  const { data: refData } = await octokit.git.getRef({
    owner,
    repo,
    ref: `heads/${branch}`,
  });
  const currentCommitSha = refData.object.sha;

  // ═══════════════════════════════════════════════════════════════════
  // STEP 2: Get the current tree SHA from that commit
  // ═══════════════════════════════════════════════════════════════════
  const { data: commitData } = await octokit.git.getCommit({
    owner,
    repo,
    commit_sha: currentCommitSha,
  });
  const currentTreeSha = commitData.tree.sha;

  // ═══════════════════════════════════════════════════════════════════
  // STEP 3: Create blobs for all new/updated files
  // ═══════════════════════════════════════════════════════════════════
  // Filter out deletions and create blobs in parallel
  const filesToCreate = operations.filter(op => op.content !== null);
  
  const blobPromises = filesToCreate.map(async (op) => {
    const content = Buffer.isBuffer(op.content)
      ? op.content.toString('base64')
      : op.content!;
    
    const encoding = Buffer.isBuffer(op.content) ? 'base64' : 'utf-8';

    const { data: blob } = await octokit.git.createBlob({
      owner,
      repo,
      content,
      encoding,
    });

    return { path: op.path, sha: blob.sha };
  });

  const blobs = await Promise.all(blobPromises);
  const blobMap = new Map(blobs.map(b => [b.path, b.sha]));

  // ═══════════════════════════════════════════════════════════════════
  // STEP 4: Build tree entries for ALL operations
  // ═══════════════════════════════════════════════════════════════════
  type TreeEntry = {
    path: string;
    mode: '100644';  // Regular file
    type: 'blob';
    sha: string | null;  // null = delete
  };

  const treeEntries: TreeEntry[] = operations.map(op => ({
    path: op.path,
    mode: '100644',
    type: 'blob',
    sha: op.content === null ? null : blobMap.get(op.path)!,
  }));

  // ═══════════════════════════════════════════════════════════════════
  // STEP 5: Create the new tree
  // ═══════════════════════════════════════════════════════════════════
  // base_tree means: "start with this tree, apply these changes"
  const { data: newTree } = await octokit.git.createTree({
    owner,
    repo,
    base_tree: currentTreeSha,  // Incremental, not full replacement
    tree: treeEntries,
  });

  // ═══════════════════════════════════════════════════════════════════
  // STEP 6: Create the commit pointing to the new tree
  // ═══════════════════════════════════════════════════════════════════
  const { data: newCommit } = await octokit.git.createCommit({
    owner,
    repo,
    message,
    tree: newTree.sha,
    parents: [currentCommitSha],
  });

  // ═══════════════════════════════════════════════════════════════════
  // STEP 7: Update the branch reference to point to new commit
  // ═══════════════════════════════════════════════════════════════════
  await octokit.git.updateRef({
    owner,
    repo,
    ref: `heads/${branch}`,
    sha: newCommit.sha,
  });

  console.log(`[GitHub] Batch commit complete: ${newCommit.sha}`);
  return newCommit.sha;
}
```

### Key Techniques Used

#### 1. Parallel Blob Creation

```typescript
// DON'T: Sequential (slow)
for (const file of files) {
  await octokit.git.createBlob({ ... }); // One at a time
}

// DO: Parallel (fast)
const blobPromises = files.map(file => 
  octokit.git.createBlob({ ... })
);
await Promise.all(blobPromises); // All at once
```

*Why it matters:* For 5 files, parallel is ~5x faster than sequential.

#### 2. Incremental Tree with base_tree

```typescript
// WITHOUT base_tree: Must include EVERY file in repo
const { data } = await octokit.git.createTree({
  owner, repo,
  tree: everyFileInRepo, // Thousands of entries!
});

// WITH base_tree: Only include changes
const { data } = await octokit.git.createTree({
  owner, repo,
  base_tree: currentTreeSha,  // "Start here..."
  tree: justTheChanges,       // "...apply these"
});
```

*Why it matters:* `base_tree` makes the operation O(changes) not O(files in repo).

#### 3. Deletion via null SHA

```typescript
// For files to delete, set sha to null
const deleteEntry = {
  path: 'images/old-photo.jpg',
  mode: '100644',
  type: 'blob',
  sha: null,  // ← This means "delete"
};
```

*Why it matters:* Deletions are included in the same atomic commit.


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Usage Example

```typescript
/**
 * Save portfolio with images - real-world usage pattern
 */
async function savePortfolio(
  portfolioData: object,
  newImages: Array<{ path: string; buffer: Buffer }>,
  deletedPaths: string[]
): Promise<string> {
  const operations: FileOperation[] = [
    // 1. Update the JSON data file
    {
      path: 'data/projects.json',
      content: JSON.stringify(portfolioData, null, 2),
    },
    
    // 2. Add new images
    ...newImages.map(img => ({
      path: img.path,
      content: img.buffer,
    })),
    
    // 3. Delete removed images
    ...deletedPaths.map(path => ({
      path,
      content: null,  // null = delete
    })),
  ];

  return batchCommit(
    octokit,
    'owner',
    'repo',
    'main',
    'Update portfolio via editor',
    operations
  );
}

// Example call
await savePortfolio(
  { title: 'My Project', description: '...' },
  [
    { path: 'images/hero.jpg', buffer: heroImageBuffer },
    { path: 'images/thumb.jpg', buffer: thumbImageBuffer }
  ],
  ['images/old-unused.jpg']
);
// Result: ONE commit with 4 file operations
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Testing Approach

```typescript
describe('batchCommit', () => {
  it('should create single commit for multiple file operations', async () => {
    // Arrange
    const operations = [
      { path: 'file1.txt', content: 'content 1' },
      { path: 'file2.txt', content: 'content 2' },
    ];

    // Act
    const commitSha = await batchCommit(
      mockOctokit,
      'owner',
      'repo',
      'main',
      'Test commit',
      operations
    );

    // Assert
    expect(mockOctokit.git.createCommit).toHaveBeenCalledTimes(1);
    expect(commitSha).toBeDefined();
  });

  it('should handle mixed create/delete operations atomically', async () => {
    // Arrange
    const operations = [
      { path: 'new.txt', content: 'new content' },
      { path: 'old.txt', content: null }, // Delete
    ];

    // Act & Assert
    const result = await batchCommit(/*...*/);
    
    // Verify tree contains create AND delete
    const treeCall = mockOctokit.git.createTree.mock.calls[0];
    expect(treeCall[0].tree).toContainEqual(
      expect.objectContaining({ path: 'new.txt', sha: expect.any(String) })
    );
    expect(treeCall[0].tree).toContainEqual(
      expect.objectContaining({ path: 'old.txt', sha: null })
    );
  });

  it('should fail atomically if any operation fails', async () => {
    // Arrange
    mockOctokit.git.createBlob.mockRejectedValueOnce(new Error('API Error'));

    // Act & Assert
    await expect(batchCommit(/*...*/)).rejects.toThrow('API Error');
    
    // Verify no partial commit occurred
    expect(mockOctokit.git.updateRef).not.toHaveBeenCalled();
  });
});
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Performance Considerations

**Before (Contents API - Per-File Commits):**
```
5 file save operation:
- API calls: 5 (one per file)
- Commits created: 5
- Total time: ~5 × 200ms = 1000ms (sequential)
- Risk: Partial failure leaves inconsistent state
```

**After (Git Trees API - Batch Commit):**
```
5 file save operation:
- API calls: 7 (parallel blobs + tree + commit + ref)
- Commits created: 1
- Total time: ~500ms (blob creation parallelized)
- Risk: Zero (atomic - all succeed or none)
```

| Metric | Per-File | Batch | Improvement |
|--------|----------|-------|-------------|
| Commits per save | N | 1 | Cleaner history |
| API calls | N | ~N+3 | Similar (but parallel) |
| Atomicity | No | Yes | Data integrity |
| Rollback complexity | High | Low | Simple git revert |


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Lessons Learned

### What Worked Well
- **Atomicity guarantee** eliminates "half-saved" data states
- **Parallel blob creation** significantly reduces latency
- **base_tree parameter** makes incremental updates efficient
- **Clean commit history** aids debugging and rollbacks

### What I'd Do Differently
- Add **retry logic** for transient GitHub API failures
- Implement **chunking** for very large batches (>100 files)
- Add **progress callbacks** for UI feedback on large operations
- Consider **caching current tree SHA** to reduce API calls

### Key Takeaways
1. The Git Trees API is more powerful than the Contents API
2. Most developers don't know about `base_tree` for incremental updates
3. Atomicity should be a first-class requirement for data operations
4. Parallel operations are essential for acceptable latency


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Common Pitfalls

### 1. Forgetting base_tree
```typescript
// ❌ BAD: Creates tree with ONLY specified files
await octokit.git.createTree({
  owner, repo,
  tree: [{ path: 'new.txt', ... }],
  // Missing base_tree = all other files deleted!
});

// ✅ GOOD: Applies changes on top of existing tree
await octokit.git.createTree({
  owner, repo,
  base_tree: currentTreeSha,  // Preserve existing files
  tree: [{ path: 'new.txt', ... }],
});
```

### 2. Not Handling Binary Files
```typescript
// ❌ BAD: Sending binary as string
createBlob({ content: imageBuffer, encoding: 'utf-8' });

// ✅ GOOD: Base64 encode binary
createBlob({ 
  content: imageBuffer.toString('base64'), 
  encoding: 'base64' 
});
```


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Related Patterns

- [Edge Middleware Auth](./edge-middleware-auth.md) - How requests are authenticated
- [Pending Media Context](../frontend/pending-media-context.md) - Client-side staging
- [Hybrid Data Storage](../patterns/hybrid-data-storage.md) - Why GitHub is the data store


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


## Further Reading

- [GitHub Git Data API](https://docs.github.com/en/rest/git)
- [Octokit.js Documentation](https://octokit.github.io/rest.js/)
- [Git Internals - Pro Git Book](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects)
- [GitHub App Authentication](https://docs.github.com/en/apps/creating-github-apps)


<div align="center">
<img width="600" src="https://raw.githubusercontent.com/andreasbm/readme/master/assets/lines/aqua.png"/>
</div>


*Sample complexity: Very High | Education value: Very High | Interview relevance: High*
