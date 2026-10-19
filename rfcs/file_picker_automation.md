# RFC: File and directory picker automation in WPT

## Summary

Add `testdriver.js` support for automating file and directory picker dialogs triggered by the [File System Access API](https://wicg.github.io/file-system-access/) (`window.showOpenFilePicker()`, `window.showSaveFilePicker()`, and `window.showDirectoryPicker()`).


## Details

Currently, tests under `file-system-access/` that invoke file or directory pickers must be marked as manual tests because JavaScript running in automated test environments cannot interact with platform-native OS dialogs. This prevents automated cross-browser CI coverage for picker APIs and downstream filesystem functionality (including the 26 manual local filesystem tests under `file-system-access/local_*.https.html`).

This RFC proposes adding an ahead-of-time (AOT) pre-configuration API to `testdriver.js`, paired with runner-side path resolution and temporary directory provisioning:

### **1. File Picker Automation API (`testdriver.js`)**

* `test_driver.set_file_picker_behavior(options, context = null)`: Pre-configures the behavior of the next file or directory picker (one-shot).
  * `options.action`: `"accept"` or `"cancel"`.
  * `options.paths`: An array of relative paths to be selected when `action` is `"accept"`, and disallowed when `action` is `"cancel"`.
  * **Acceptance (`{ action: "accept", paths: [...] }`)**: Pre-configures the paths to return. Passing an empty array (`[]`), non-array, or omitting paths throws an upfront `TypeError`, as confirming zero files is not a valid OS dialog state. Providing multiple paths to a single-select picker (e.g. `showOpenFilePicker({ multiple: false })` or `showSaveFilePicker()`) causes the test harness to fail at dialog invocation.
  * **Cancellation (`{ action: "cancel" }`)**: Simulates user cancellation or dismissing the dialog, causing the next picker call to reject with an `AbortError`.

### **2. Path Resolution & Temporary Directory Provisioning (`wptrunner`)**

Because `testdriver.js` is intended for user/browser interaction rather than OS filesystem management, and to avoid exposing host OS absolute paths to JavaScript tests, tests pass **relative paths** to `set_file_picker_behavior` while `wptrunner` resolves them to absolute host paths:

* Following the established WebExtensions pattern in `wptrunner` (using `testdriver.js?feature=...` or path matching such as `file-system-access/local_*`), `wptrunner` automatically provisions an isolated writable directory in the system temporary folder (via Python's `tempfile` module) before the test starts and resolves relative paths (e.g. `paths: ["."]`) under that temporary directory.
* For read-only tests that do not request a writable temporary directory, relative paths are resolved against the test's source directory (matching `WebExtensionsProtocolPart._resolve_path`).
* Deterministic cleanup of temporary directories is handled by `wptrunner` upon test completion or runner exit, preventing source tree corruption, parallel-run flakiness, and disk leaks even if browser tests crash.


## Protocol & Runner Integration

* Cross-browser execution maps cleanly onto the proposed WebDriver BiDi **`browser.setFileDialogBehavior`** command ([w3c/webdriver-bidi#1177](https://github.com/w3c/webdriver-bidi/issues/1177)), where `{ action: "accept", paths }` (after `wptrunner` resolves relative paths to absolute host paths) maps to `type: "allowed"` with `files`, and `{ action: "cancel" }` maps to `type: "denied"`.
* In-browser test harnesses can also delegate directly via `testdriver-vendor.js` to internal mock dialog APIs.


Links:

* [WebDriver BiDi Issue 1177 (`browser.setFileDialogBehavior`)](https://github.com/w3c/webdriver-bidi/issues/1177)
* [Prototype Implementation in Chromium](https://chromium-review.googlesource.com/c/chromium/src/+/8482432) (to be updated)


## Risks

* **API Surface:** Minimal impact on `testdriver.js`, adding only a single extensible `set_file_picker_behavior` method while keeping temporary directory provisioning and path resolution internal to `wptrunner`.
* **Protocol Alignment:** Requires standardizing the ahead-of-time file dialog behavior command in WebDriver BiDi (tracked in [w3c/webdriver-bidi#1177](https://github.com/w3c/webdriver-bidi/issues/1177)).
* **Remote / Mobile Environments:** Runner-side temporary directory provisioning and path resolution are scoped to co-located desktop CI where the runner and browser share the filesystem; remote device bridging (e.g. Android) can be added as a future transport extension.
