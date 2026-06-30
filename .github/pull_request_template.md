<!--
Optional linked context:
Add a visible `Closes #<issue-number>` or `Related: #<issue-number>` line
below this comment.

Required PR title:
type: user-facing description
Use a parenthesized scope only when it adds clarity:
fix(auth): login redirect loops when session cookie is expired

Types: feat, fix, improve, refactor, docs, chore.
For fixes, describe the user-visible symptom and trigger:
fix: task list fails to load when user has no environments
Avoid implementation details such as:
fix: add null check to task query
-->

## What Problem This Solves

Fixes an issue where media generation tasks (image, video, music) would fail to deliver completed results to users when the background task completion wake mechanism could not confirm delivery. Even though the media was successfully generated and saved to disk, users would not receive their content if the subagent announce path lost the handoff or returned `false` without throwing an error.

Users requesting media generation through detached background tasks would experience silent failures where the task appeared to complete but no result was delivered to the chat channel.

## Why This Change Was Made

The fix extends the existing direct channel delivery fallback pattern—already used when the wake path throws an error—to also handle cases where `wakeTaskCompletion` returns `false` (delivery not confirmed). This ensures consistent recovery regardless of how the wake path signals failure.

Key changes:

- `media-generate-background-shared.ts`: Adds a direct delivery attempt in the "delivery not confirmed" branch, using the same pattern as the catch branch for thrown errors
- `media-generate-background-shared.test.ts`: Adds focused test coverage proving the fallback recovers when wake returns `false` with a deliverable `requesterOrigin`

This approach reuses proven delivery code rather than introducing new mechanisms, keeping the fix minimal and reliable. The idempotency key uses a distinct `"blocked"` suffix to prevent duplicate sends while ensuring the user receives their generated content.

## User Impact

Users will now reliably receive their generated media (images, videos, music) even when the background task completion wake mechanism encounters delivery issues. Previously, certain failure modes would leave users without their content despite successful generation.

No action required from users—this is a transparent reliability improvement. No config changes or new surfaces; the fix operates entirely within the existing task completion and delivery infrastructure.

## Evidence

**New test coverage** (`src/agents/tools/media-generate-background-shared.test.ts`):

```typescript
it("recovers via direct fallback when wake returns false with deliverable requesterOrigin", async () => {
  // Test verifies:
  // 1. Direct delivery is attempted when wakeTaskCompletion returns false
  // 2. sendMessage is called with correct channel, target, content, and idempotency key
  // 3. terminalResult is cleared when direct delivery succeeds
});
```

**Behavior proof**: The fix mirrors the existing catch-branch pattern that handles thrown errors, extending it to the `false` return case. Both paths now attempt direct channel delivery before marking the task as terminally failed, ensuring users see their generated content regardless of how the wake path fails.

**Scope validation**: The change is narrowly scoped to the completion handler's "delivery not confirmed" branch. It does not alter:

- Primary announce delivery path
- Task lifecycle tracking or progress updates
- Visible-reply contracts or message tool usage
- Normal success paths where delivery is confirmed

**Risk assessment**:

- Minimal risk: reuses proven direct delivery code already present in the catch branch
- Idempotency preserved: distinct `idempotencySuffix: "blocked"` prevents duplicate sends
- No config surface changes: purely internal reliability improvement
- Backward compatible: only affects failure recovery, not normal operation

**Test suite**: All existing tests pass; the new test provides targeted coverage for this specific recovery scenario.
