# RFC: File and directory picker automation in WPT

## Summary

Add `testdriver.js` support for automating file and directory picker dialogs triggered by the [File System Access API](https://wicg.github.io/file-system-access/) (`window.showOpenFilePicker()`, `window.showSaveFilePicker()`, and `window.showDirectoryPicker()`).


## Details

Currently, tests under `file-system-access/` that invoke file or directory pickers must be marked as manual tests because JavaScript running in automated test environments cannot interact with platform-native OS dialogs. This prevents automated cross-browser CI coverage for picker APIs and downstream filesystem functionality (including the 26 manual local filesystem tests under `file-system-access/local_*.https.html`).

This RFC proposes adding an ahead-of-time (AOT) pre-configuration API to `testdriver.js`, paired with runner-side path resolution and temporary directory provisioning:

### **1. File Picker Automation API (`testdriver.js`)**

* `test_driver.set_file_picker_behavior(options)`: Pre-configures the behavior of the next file or directory picker (one-shot).
  * `options.action`: `"accept"` or `"cancel"`.
  * `options.paths`: An array of relative paths to be selected when `action` is `"accept"`, and disallowed when `action` is `"cancel"`.
  * **Acceptance (`{ action: "accept", paths: [...] }`)**: Pre-configures the paths to return. Passing an empty array (`[]`), non-array, or omitting paths throws an upfront `TypeError`, as confirming zero files is not a valid OS dialog state. Providing multiple paths to a single-select picker (e.g. `showOpenFilePicker({ multiple: false })` or `showSaveFilePicker()`) cancels the dialog, causing the picker call to reject with an `AbortError`.
  * **Cancellation (`{ action: "cancel" }`)**: Simulates user cancellation or dismissing the dialog, causing the next picker call to reject with an `AbortError`.

### **2. Path Resolution & Temporary Directory Provisioning (`wptrunner`)**

Because `testdriver.js` is intended for user/browser interaction rather than OS filesystem management, and to avoid exposing host OS absolute paths to JavaScript tests, tests pass **relative paths** to `set_file_picker_behavior` while `wptrunner` resolves them to absolute host paths:

* By default, relative paths (e.g. `paths: ["resources/data/testfile.txt"]`) are resolved against the test's source directory (matching `WebExtensionsProtocolPart._resolve_path`), allowing read-only tests to access existing test files without any extra configuration.
* For tests that need a writable directory, `wptrunner` provisions an isolated directory in the system temporary folder (via Python's `tempfile` module), preventing source tree corruption and parallel-run flakiness. Two mechanisms are considered for triggering this:
  * **Path prefix**: Any path starting with `"writable/"` (e.g. `paths: ["writable/"]` or `paths: ["writable/testfile.txt"]`) causes `wptrunner` to create the temporary directory on demand and resolve the path under it.
  * **Test-level declaration**: Following the WebExtensions pattern in `wptrunner`, tests specify `testdriver.js?feature=createTempDirectory` (or a `META` tag), which provisions the temporary directory before the test starts and resolves relative paths (e.g. `paths: ["."]`) under it.
* Deterministic cleanup of temporary directories is handled by `wptrunner` upon test completion or runner exit, preventing disk leaks even if browser tests crash.

## Protocol & Runner Integration

* Cross-browser execution maps cleanly onto the proposed WebDriver BiDi **`browser.setFileDialogBehavior`** command ([w3c/webdriver-bidi#1177](https://github.com/w3c/webdriver-bidi/issues/1177)), where `{ action: "accept", paths }` (after `wptrunner` resolves relative paths to absolute host paths) maps to `type: "allowed"` with `files`, and `{ action: "cancel" }` maps to `type: "denied"`.
* In-browser test harnesses can also delegate directly via `testdriver-vendor.js` to internal mock dialog APIs.


Links:

* [WebDriver BiDi Issue 1177 (`browser.setFileDialogBehavior`)](https://github.com/w3c/webdriver-bidi/issues/1177)
* [Prototype Implementation in Chromium](https://chromium-review.googlesource.com/c/chromium/src/+/8482432) (to be updated)


## Risks

* **API Surface:** Minimal impact on `testdriver.js`, adding only a single extensible `set_file_picker_behavior` method while keeping temporary directory provisioning and path resolution internal to `wptrunner`.
* **Protocol Alignment:** Requires standardizing the ahead-of-time file dialog behavior command in WebDriver BiDi (tracked in [w3c/webdriver-bidi#1177](https://github.com/w3c/webdriver-bidi/issues/1177)).
