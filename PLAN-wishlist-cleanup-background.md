# Plan: Move Wishlist Cleanup to Background Task with Progress

## Problem

The wishlist cleanup (`POST /api/wishlist/cleanup`) runs synchronously in the Flask request handler. For large wishlists, the UI shows a loading overlay with no progress info and blocks all interaction until it finishes. It could also time out for very large wishlists.

## Current State

### Backend (web_server.py:25431)
- Blocking synchronous loop: iterates every wishlist track, calls `db.check_track_exists()` per track, removes matches via `wishlist_service.mark_track_download_result()`.
- Returns `removed_count` and `processed_count` only after all work is done.
- A near-identical function `_automatic_wishlist_cleanup_after_db_update()` (web_server.py:22139) already runs in a background thread via `missing_download_executor.submit()` -- but has no progress reporting.

### Frontend (script.js)
- Two callers:
  1. `cleanupWishlistOverview()` (script.js:14130) -- shows `showLoadingOverlay()`, awaits response, hides overlay.
  2. `cleanupWishlist(playlistId)` (script.js:29519) -- disables button, awaits response, re-enables.
- Neither shows progress (track count, percentage, current track).

## Existing Infrastructure to Reuse

### 1. Automation Progress System (best fit)
- **Backend**: `_init_automation_progress(id, name, action_type)`, `_update_automation_progress(id, progress=, phase=, processed=, total=, log_line=, status=)` -- thread-safe state dict emitted via `socketio.emit('automation:progress', ...)` every ~1s by `_emit_automation_progress_loop()`.
- **Frontend**: `socket.on('automation:progress', updateAutomationProgressFromData)` renders a progress bar, phase text, and scrolling log inside automation cards.
- **Issue**: This is tightly coupled to automation cards on the Automations page. Wishlist cleanup is a manual user action, not an automation. Would need a separate UI panel or a reusable progress component.

### 2. Socket.IO Custom Event (simpler, more appropriate)
- Emit a dedicated `wishlist:cleanup_progress` event from the background thread.
- Frontend listens for it and updates a lightweight progress indicator (inline in the modal, not a full automation card).
- Pattern already used: `wishlist:stats` is emitted for auto-processing state.

### 3. ThreadPoolExecutor
- `missing_download_executor` (max_workers=3) is the standard executor for wishlist/download work. Already used for the auto-cleanup.

### 4. Toast + Notification System
- `showToast(msg, type)` for completion/error notifications.
- Already used by both cleanup callers.

## Proposed Approach

Use **Socket.IO custom event + background thread** (option 2+3). This is the simplest path that fits the existing patterns.

### Backend Changes (web_server.py)

1. **New cleanup state dict** (similar pattern to `db_update_state`):
   ```python
   wishlist_cleanup_state = {
       "running": False,
       "processed": 0,
       "total": 0,
       "removed": 0,
       "current_track": "",
   }
   wishlist_cleanup_lock = threading.Lock()
   ```

2. **Convert the `/api/wishlist/cleanup` endpoint** to:
   - Check if cleanup is already running; if so, return `{"success": false, "error": "Cleanup already in progress"}`.
   - Submit `_run_wishlist_cleanup(profile_id)` to `missing_download_executor`.
   - Return immediately with `{"success": true, "started": true}`.

3. **New `_run_wishlist_cleanup(profile_id)` function**:
   - Set `wishlist_cleanup_state` to running, reset counters.
   - Iterate tracks (same logic as current endpoint).
   - After each track, update `wishlist_cleanup_state` with processed/total/removed/current_track.
   - On completion, set running=False.
   - Emit `socketio.emit('wishlist:cleanup_progress', wishlist_cleanup_state)` after each track (or batch of ~5 tracks to reduce chatter).

4. **Add to the existing emit loop** (`_emit_download_status_loop` or similar):
   - While cleanup is running, emit `wishlist:cleanup_progress` every 1-2s with current state. This avoids per-track emit overhead and piggybacks on existing infrastructure.

5. **New status endpoint** `GET /api/wishlist/cleanup/status`:
   - Returns current `wishlist_cleanup_state`. Used for polling fallback / initial state on page load.

6. **Deduplicate**: Refactor `_automatic_wishlist_cleanup_after_db_update()` to call the same `_run_wishlist_cleanup()` core logic (just without the Socket.IO progress -- or with it, since auto-cleanup progress is useful too).

### Frontend Changes (script.js)

1. **Register Socket.IO listener**:
   ```javascript
   socket.on('wishlist:cleanup_progress', (data) => updateWishlistCleanupProgress(data));
   ```

2. **New `updateWishlistCleanupProgress(data)` function**:
   - If `data.running`: show/update an inline progress indicator in the currently open modal (or a persistent toast/banner if no modal is open).
     - Display: "Checking track 42/150... (3 removed so far)"
     - Optional: slim progress bar.
   - If `!data.running` and was previously running: show completion toast, refresh wishlist stats, close progress indicator.

3. **Update `cleanupWishlistOverview()`**:
   - After confirm, POST to `/api/wishlist/cleanup`.
   - On success response (`started: true`): swap the loading overlay for the progress indicator. Close modal or keep it open with progress -- user can navigate away.
   - Remove `showLoadingOverlay()` / `hideLoadingOverlay()`.

4. **Update `cleanupWishlist(playlistId)`**:
   - Same pattern: fire-and-forget POST, show progress inline in the button area.
   - On socket completion event: re-enable button, show toast, refresh modal.

5. **Page load / modal open**: Check `GET /api/wishlist/cleanup/status` to pick up in-progress cleanup (e.g., user refreshed page).

## Implementation Order

1. Add `wishlist_cleanup_state` dict and lock.
2. Extract shared cleanup logic into `_run_wishlist_cleanup()`.
3. Convert API endpoint to submit-and-return.
4. Add progress emission to emit loop.
5. Add `GET /api/wishlist/cleanup/status` endpoint.
6. Frontend: add socket listener and progress UI.
7. Frontend: update both `cleanupWishlistOverview()` and `cleanupWishlist()`.
8. Test with a non-trivial wishlist.

## Scope / Non-Goals

- No new dependencies (no Celery, no Redis).
- No changes to the auto-cleanup-after-db-update behavior beyond deduplicating the core logic.
- No new modal -- reuse existing UI space in the wishlist modals.
- Keep the loading overlay pattern for the "Clear Wishlist" action (that one is fast, just a DB delete).
