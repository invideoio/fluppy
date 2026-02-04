# Remove Completer from MultipartUploadController

**Date:** 2026-02-04
**File:** `lib/src/s3/multipart_upload_controller.dart`

---

## Problem

When `cancel()` was called externally while `start()` was suspended on `await _startUpload()`, the `_completer.completeError()` fired before any listener existed on `_completer.future`. Dart treats this as an unhandled async error.

A previous attempt fixed this with `_completer.future.ignore()` in the constructor, but that was a band-aid — it suppressed *all* errors on the completer's future, masking real bugs.

The root cause was that `Completer` was unnecessary indirection. `start()` already awaited `_startUpload()` / `_resumeUpload()` synchronously, so results and errors could flow through normal return values and exceptions. The `Completer` created a temporal gap: an error could be pushed into it (by `cancel()`) before anyone was listening.

---

## What Changed

### 1. Replaced `Completer<UploadResponse>` with `UploadResponse? _result`

```dart
// Before
final Completer<UploadResponse> _completer = Completer();

// After
UploadResponse? _result;
```

**Why:** The completer was the source of the unhandled error. A simple nullable field caches the completed response without creating a detached Future.

### 2. Restored `continueExisting` constructor parameter

```dart
// Before (broken — parameter accepted but never used)
MultipartUploadController({
  ...
  bool continueExisting = false,
});

// After
MultipartUploadController({
  ...
  bool continueExisting = false,
}) : _uploadStarted = continueExisting;
```

**Why:** This was commented out during an earlier debugging session. Without it, the "app restart" resume path (where a new controller is created for an existing S3 upload) would call `_startUpload()` instead of `_resumeUpload()`, hitting `createMultipartUpload` when it shouldn't. Five tests depended on this behavior.

### 3. Simplified `start()` — direct return, no completer

```dart
// Before
Future<UploadResponse> start() async {
  ...
  await _startUpload();
  if (_state == UploadState.paused) throw PausedException();
  return _completer.future;
}

// After
Future<UploadResponse> start() async {
  ...
  return await _startUpload();   // or _resumeUpload()
}
```

**Why:** `_startUpload()` and `_resumeUpload()` now return `UploadResponse` directly (or throw). No need for a separate completer future or post-await state checks — `PausedException` propagates naturally through `await`.

### 4. Changed `_startUpload()` from `Future<void>` to `Future<UploadResponse>`

Key differences from the old version:
- **Returns** `UploadResponse` instead of completing a completer
- **Throws** `PausedException()` on pause instead of silently returning
- **Guards** against overwriting `cancelled` state with `error` (`if (_state != UploadState.cancelled)`)
- Uses `rethrow` instead of `_completer.completeError(e)`

**Why:** Standard Dart async/await pattern. Errors flow through `rethrow` up the call stack instead of being pushed into a detached completer.

### 5. Changed `_resumeUpload()` from `Future<void>` to `Future<UploadResponse>`

Same changes as `_startUpload()`. Also returns cached `_result` if already completed.

### 6. Simplified `cancel()`

```dart
// Before
void cancel() {
  if (_state == UploadState.completed) return;
  _state = UploadState.cancelled;
  _cancelToken?.cancel();
  if (!_completer.isCompleted) {
    _completer.completeError(CancelledException("Rohan 3"));
  }
}

// After
void cancel() {
  if (_state == UploadState.completed) return;
  _state = UploadState.cancelled;
  _cancelToken?.cancel();
}
```

**Why:** No completer to push errors into. The `_cancelToken?.cancel()` causes in-flight Dio operations to throw `DioException(type: cancel)`, which propagates up through `_uploadPartData()` → `_uploadParts()` → `_startUpload()` → `start()` → caller. The `_state = UploadState.cancelled` is picked up by `_throwIfCancelled()` on the next checkpoint.

### 7. Added paused state check to `_throwIfCancelled()`

```dart
// Added this block
if (_state == UploadState.paused) {
  throw CancelledException(_pausingReason);
}
```

**Why:** `pause()` cancels the current `_cancelToken` but then immediately replaces it with a fresh one (for the future resume). This means `_throwIfCancelled()` couldn't detect a pause by checking `_cancelToken.isCancelled` — the new token isn't cancelled. Adding a `_state == UploadState.paused` check closes this gap. This was a pre-existing bug that was masked by the old completer-based approach.

### 8. Updated test: "resume throws PausedException if paused again during resume"

- The initial `uploadFuture` now throws `PausedException` directly (instead of eventually completing via the completer), so the test awaits it immediately after the pause
- Increased `listParts` delay from 150ms to 300ms for timing reliability

---

## Why This Is Better

| Aspect | Before (Completer) | After (Direct Return) |
|--------|--------------------|-----------------------|
| Error path | Two parallel paths: completer + natural propagation | Single path: normal `rethrow` |
| Temporal safety | Error can fire before listener exists | No detached Future — errors propagate through `await` |
| Workarounds | `_completer.future.ignore()` needed | None needed |
| `cancel()` complexity | Must push error into completer | Just set state + cancel token |
| Code pattern | Non-standard Dart (Completer bridge) | Standard `async/await` return/throw |

---

## Files Modified

- `lib/src/s3/multipart_upload_controller.dart` — core changes
- `test/s3_uploader_test.dart` — one test updated for new behavior

## Verification

- All 149 tests pass (`dart test`)
- `dart analyze` clean on modified files (pre-existing warnings in other files unchanged)
