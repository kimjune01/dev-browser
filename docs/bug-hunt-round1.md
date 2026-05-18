# Bug Hunt Round 1

## Finding 1

- Severity: high
- File: [daemon/src/sandbox/quickjs-sandbox.ts](/Users/junekim/Documents/dev-browser/daemon/src/sandbox/quickjs-sandbox.ts:422), [daemon/src/sandbox/host-bridge.ts](/Users/junekim/Documents/dev-browser/daemon/src/sandbox/host-bridge.ts:54), [README.md](/Users/junekim/Documents/dev-browser/README.md:181)
- Bug: the sandbox exposes raw Playwright client objects to guest scripts and forwards arbitrary Playwright protocol messages to the host without an allowlist. That breaks the repo’s “no host fs” contract because many normal Playwright APIs accept host file paths.
- Description: `waitForConnectionObject()` returns unrestricted Playwright `Page` objects into the QuickJS sandbox, and `HostBridge.receiveFromSandbox()` dispatches any JSON protocol message into the host-side Playwright dispatcher. The README and CLI help explicitly advertise that these are “full Playwright Page objects”. That makes path-based Playwright features reachable from the sandbox even though the custom `writeFile`/`readFile` helpers are restricted.
- Trigger / attack scenario: a sandboxed script can read arbitrary host files with Playwright’s upload path APIs, for example:

```js
const page = await browser.newPage();
await page.setContent('<input type="file" id="f">');
await page.setInputFiles('#f', '/etc/passwd');
const text = await page.evaluate(async () => {
  return await document.querySelector('#f').files[0].text();
});
console.log(text);
```

  It can also write arbitrary host files with Playwright path options such as `page.screenshot({ path: ... })`, `locator.screenshot({ path: ... })`, or `page.context().storageState({ path: ... })`.
- Suggested fix: do not expose raw Playwright client objects directly. Wrap or filter the API surface so only vetted methods are available, and explicitly reject any Playwright call that accepts a host path or other host-side resource handle. If keeping the protocol bridge, add a method/parameter allowlist on the host side.

## Finding 2

- Severity: medium
- File: [cli/src/daemon.rs](/Users/junekim/Documents/dev-browser/cli/src/daemon.rs:35), [daemon/src/daemon.ts](/Users/junekim/Documents/dev-browser/daemon/src/daemon.ts:395)
- Bug: daemon startup is not serialized, so concurrent CLI invocations can spawn multiple daemons that race on `daemon.sock` and `daemon.pid`.
- Description: `ensure_daemon()` only probes the socket and then spawns a daemon if the socket is absent. There is no lockfile or atomic claim step. On the daemon side, startup unconditionally unlinks the socket path, writes the pid file, and only then starts listening. Two `dev-browser` commands started at nearly the same time can both decide the daemon is absent and both spawn.
- Trigger / attack scenario: run two `dev-browser` commands in parallel on a cold start. One daemon can unlink the other daemon’s bound socket path and overwrite `daemon.pid`, leaving an orphaned background process that still owns browser instances but is no longer discoverable by future clients. On Unix, unlinking a live Unix socket path does not stop the already-bound server; it only removes the name. A second daemon can then bind the same pathname and split future clients from the original process.
- Suggested fix: add a real startup lock around daemon creation, or atomically create and hold a lockfile before spawning. The daemon should also avoid unlinking a socket path that might belong to a live process without verifying ownership.

## Finding 3

- Severity: medium
- File: [daemon/src/daemon.ts](/Users/junekim/Documents/dev-browser/daemon/src/daemon.ts:129), [daemon/src/daemon.ts](/Users/junekim/Documents/dev-browser/daemon/src/daemon.ts:315)
- Bug: `execute` requests are serialized per browser, but `browser-stop` is not, so a second client can tear down a browser while a script is actively using it.
- Description: `handleExecute()` runs under `withBrowserLock(request.browser, ...)`, but `browser-stop` calls `manager.stopBrowser()` directly with no matching lock. That means lifecycle operations on the same browser are not actually mutually exclusive.
- Trigger / attack scenario: client A starts a long-running script against browser `default`; client B sends `browser-stop default` while the script is midway through Playwright RPCs. The manager removes the browser entry and closes contexts/pages underneath the active QuickJS sandbox. The result is nondeterministic failures, partial cleanup, and potential follow-on errors while the sandbox still has live Playwright object references.
- Suggested fix: run `browser-stop` under the same keyed browser lock as `execute`, and consider using the same lock for any operation that mutates browser lifecycle state.

## Finding 4

- Severity: low
- File: [daemon/src/daemon.ts](/Users/junekim/Documents/dev-browser/daemon/src/daemon.ts:412)
- Bug: the daemon accepts newline-delimited JSON over a local socket but imposes no maximum request size, so a client can cause unbounded memory growth by streaming a partial line.
- Description: each socket appends incoming data into `buffer` until a newline arrives, with no cap on line length, connection age, or total buffered bytes. A local client does not need to send valid JSON or complete the line to consume memory.
- Trigger / attack scenario: connect to `~/.dev-browser/daemon.sock` and continuously send bytes without `\n`. The daemon keeps concatenating to `buffer` and eventually exhausts memory or becomes unstable.
- Suggested fix: enforce a maximum frame size and close the connection once the buffered request exceeds it. The same limit should apply before JSON parsing so malformed oversized frames cannot force allocation spikes.
