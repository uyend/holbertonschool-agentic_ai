# Vague prompt, two models

## Request

> Create an function that saves data and handle errors

No language, storage target, data shape, caller, or threat model was specified.

## Models

1. Claude Haiku 4.5
2. Claude Opus 5.5

## Observed differences

Haiku asked four clarifying questions (data type, language, destination, error classes) and then answered anyway with a ~60-line Python function. It assumes JSON files on a local filesystem, configures process-wide logging, catches several exception types, returns a status dictionary instead of raising, and appends a `__main__` demo that writes a sample user record.

Opus asked nothing up front. It stated its assumptions after the fact ("You didn't name a language or a storage target... so I made it a standalone Python function that writes to a local file") and delivered a more engineered result: an atomic write (temporary file, `fsync`, then `os.replace`), a single custom exception `SaveDataError` that keeps the original exception as its cause, an `overwrite` flag, and a four-test `unittest` suite it reports as passing.

Both models picked the same architecture on their own: Python, JSON, local filesystem. They differ in depth and error contract. Haiku hides failures in a `{"success": False}` dictionary and clarifies first. Opus raises exceptions, protects against half-written files, and clarifies only after it has built something. Opus's answer is higher quality, but it is also more convincing, which makes its unrequested choices easier to accept without review.

## Problems and arbitrary choices

**Haiku catches an exception `json.dump` does not raise.** Non-serializable objects make `json.dump` raise `TypeError`. `json.JSONDecodeError` comes from parsing (`json.load` / `json.loads`). The branch that reports "Data is not JSON serializable" is dead code; that failure falls through to the generic `Exception` handler. The file is also opened with `'w'` before serialization, so a failed write truncates an existing file. `logging.basicConfig` at import time mutates global logging. The file is opened without `encoding="utf-8"`, so output depends on the platform locale.

**Opus's `overwrite=False` check is not atomic.** `target.exists()` is checked before the write, but `os.replace` overwrites unconditionally. Another process can create the file in between, and it will be silently replaced. A true "do not overwrite" guarantee needs `os.link` or `open(..., "x")` semantics. The default `overwrite=True` is destructive, and nobody asked for that default.

**Opus silently changes file permissions.** `tempfile.mkstemp` creates the file with mode `0600`. After `os.replace`, an existing `0644` file ends up `0600`, readable only by its owner, which can break other services that read it. The directory is also not `fsync`ed after the rename, so the claim that the write survives a crash is not fully guaranteed on every filesystem.

Shared arbitrary choices: both models chose Python, JSON, and the local filesystem without being asked. Both create missing parent directories (`mkdir(parents=True)`), so any path the caller passes, including one built from user input such as `../../etc/...`, is created and written without validation. Neither limits payload size, and both log the full target path, which may expose sensitive locations in logs.

## Self-assessment

**Haiku**

- Invented a failure mode: `json.dump` never raises `JSONDecodeError`, so its "bad data" handler is dead code.
- Added an unrequested filesystem-plus-logging design.
- Its return-a-dictionary contract lets callers ignore failures.

**Opus**

- Did not hallucinate APIs, but made unrequested decisions: atomic replace, a destructive `overwrite=True` default, automatic directory creation, and a custom exception hierarchy.
- Missed a time-of-check/time-of-use race in its `overwrite=False` check.
- Missed that the `0600` temp-file mode changes permissions on existing files.

**Both**

- Neither validates paths, so `../../etc/...` is accepted.
- Neither considers who may write where, or limits payload size.
- Polished output and passing tests make unstated requirements easy to approve.

**Takeaway**

- A vague prompt makes the model fill gaps with confident choices about language, persistence, defaults, and trust boundaries.
- Tests only check choices the model made for itself.
- State concurrency, permissions, validation, and storage location up front.
