# Orca architecture review and Shepherd plan

Date: 2026-09-03

Reviewed snapshots:

- Shepherd [`8282abde`](https://github.com/jhartzell/shepherd/tree/8282abde021814af6e29057aefa01c82d2c46c56)
- Orca [`67e22345`](https://github.com/stablyai/orca/tree/67e22345daf882190911355eed152ef33e051c5e)

This review answers two questions: which Orca ideas can improve Shepherd's terminal performance, and how Shepherd can gain an iPhone or iPad client without replacing its native Swift architecture.

The recommendation is to keep Shepherd's product model and most of its package graph. Build on its existing remote protocol, split control traffic from terminal traffic, add flow control, and make the host runtime callable without a SwiftUI view model. Do not port Orca's Electron, React Native, xterm, relay operations, or daemon graph.

## Scope and confidence

Orca claims below come from its source, first-party documentation, manifests, and tests at the pinned commit. Shepherd claims come from the pinned repository source and tests. Proposed performance gains are hypotheses until the benchmark plan measures them.

## Executive decisions

| ID | Decision | Reason |
| --- | --- | --- |
| D1 | Keep SwiftUI, Ghostty, SwiftTerm, `ShepherdCore`, and `ShepherdProtocol`. | Orca's useful work is in scheduling, flow control, and runtime ownership. Its UI stack does not solve a Shepherd problem. |
| D2 | Ship direct LAN or Tailscale mobile access before a cloud relay. | Shepherd already assumes a private transport path. A relay adds authentication, encryption, routing, operations, and incident response. |
| D3 | Give the host runtime a small transport-neutral interface before extracting a process. | The current remote listener still calls `ShepherdViewModel` for pane and agent creation. A headless executable cannot depend on a Mac window. |
| D4 | Keep structured control messages, but add a binary terminal stream with sequence numbers and acknowledgements. | NDJSON is clear for low-rate mutations. Base64 JSON is wasteful and hard to flow-control for terminal bytes. |
| D5 | Make mobile viewers observers by default. | A phone must not shrink the PTY used by the host or another desktop. Viewport control needs an explicit claim. |
| D6 | Preserve app-scoped PTY lifetime in the first release. | Restart survival is useful, but it changes a deliberate Shepherd contract and requires separate supervision. Add it only after headless hosting works and the need is measured. |
| D7 | Build a focused mobile companion, not desktop parity. | The first useful scope is status, recent output, prompt replies, notifications, and agent creation. Repository administration can wait. |

## Architecture comparison

### Runtime ownership

Shepherd has one in-process `SessionServer`. It owns workspace state, PTYs, screen snapshots, local extension IPC, and the optional remote listener. Every PTY queue targets the server's serial queue, which makes snapshot and live-output ordering easy to reason about. Quitting the app ends all child processes. This is a small and clear ownership model.

Orca splits runtime ownership. Its main runtime owns RPC, worktrees, persistence, and client coordination. A separately forked terminal daemon owns local PTYs. Orca can also run headlessly. A successor runtime can adopt a live terminal daemon after a process-scoped restart, but Orca's own operations guide warns that process detachment does not survive a destructive systemd cgroup stop without separate supervision.[^orca-orcad] This buys continuity but introduces daemon health, adoption, incarnation, migration, and shutdown contracts.

**Takeaway for Shepherd:** headless hosting and daemon-owned PTYs are separate decisions. Build the former first. Shepherd can run a host process without promising that PTYs survive its restart.

### Terminal model and replay

Both applications keep an authoritative host-side terminal model, then attach views using a snapshot followed by live output.

Shepherd feeds every PTY byte into `SessionScreen`, a headless SwiftTerm terminal. It builds a self-contained ANSI snapshot with up to 2,000 scrollback lines. Local attach registration, snapshot creation, and the output watermark occur in one server-queue turn.[^shepherd-screen][^shepherd-output] A local Ghostty surface then parses the snapshot and live bytes for display.[^shepherd-terminal-store]

Orca's daemon feeds output through an ordered pipeline into a headless xterm model and attached clients. Snapshots include terminal content, dimensions, modes, metadata, and an absolute output sequence. The daemon drains pending output before taking a snapshot so clients can replace an acknowledged range and deduplicate the live tail.[^orca-output-plane]

**Takeaway for Shepherd:** retain the host snapshot model and atomic attach rule. Those are strong. Add stream generations and byte positions when terminal transport becomes reconnectable and flow-controlled.

### Output scheduling and backpressure

Shepherd's local path already has the right basic shape. PTY reads are bounded. Each session allows one main-queue delivery in flight. Pending and in-flight bytes count toward a 4 MiB high-water mark, PTY reads suspend, and they resume below 1 MiB.[^shepherd-output][^shepherd-output-tests]

The remote path is weaker. The server base64-encodes terminal chunks in JSON and keeps only a per-connection write-queue limit. The client decodes every frame and enqueues one independent main-queue closure per output frame.[^shepherd-remote-server][^shepherd-remote-client] There is no per-stream acknowledgement window after bytes leave the socket.

Orca adds several controls:

- daemon output batching uses a 2 ms window and a 128 KiB shallow socket gate so bulk output does not bury interactive echo;[^orca-daemon-batcher]
- the producer can pause PTY reads, with a five-second failsafe if a resume notification is lost;[^orca-producer-pause]
- remote terminal streams use ACK windows and replace overflowed pending output with a recovery snapshot;[^orca-flow-control]
- renderer parsing prioritizes foreground terminals and stops each drain turn after a time budget;[^orca-renderer-drain]
- mobile gives an idle terminal immediate delivery, then coalesces burst tails into a 48 ms window with a 512 KiB safety cap.[^orca-mobile-coalescer]

**Takeaway for Shepherd:** copy the policy, not the implementation. Shepherd needs bounded client delivery, visible-session priority, acknowledgements, and snapshot recovery. It does not need Orca's TypeScript pipeline.

### Hidden terminal lifetime

Shepherd keeps every agent, inspector, and global-shell layout mounted. Space shells stay mounted after first use. Opacity changes visibility, and Ghostty occlusion stops hidden display links.[^shepherd-mounting] This avoids the replay flash that previously made switching slow, but the hidden Ghostty terminal still receives and parses output while the host-side SwiftTerm model parses the same bytes. Surface and scrollback memory also grow with every mounted pane.

Orca uses hysteresis. It cold-parks hidden worktrees and tabs after 30 seconds, retains a small recently used set for five minutes, and caps that set at four worktrees and six tabs.[^orca-parking]

**Takeaway for Shepherd:** do not revert to unmount-on-every-switch. Add cold parking only after measurement, with a warm recent set and an existing host snapshot for restoration.

### Remote transport and compatibility

Shepherd's control and terminal data share one ordered TCP NDJSON stream. Authentication uses one bearer token. The host requires exact protocol version equality, though individual additions already use capability strings.[^shepherd-protocol][^shepherd-handshake] The listener has no TLS and must remain behind a VPN or trusted network.

Orca uses structured RPC plus a binary multiplexed terminal stream. Optional structured fields stay optional, new opcodes require capability negotiation, and shipped opcode numbers are permanent. Orca tests current code against the newest release in both skew directions.[^orca-wire] Its phone and host can each open an outbound WebSocket to a relay that splices frames between them.[^orca-relay]

**Takeaway for Shepherd:** keep exact version rejection until mixed-version tests exist. Then move to a compatibility range plus negotiated capabilities. A later relay should carry the same host protocol and remain a transport adapter, not a second source of truth.

### Mobile product shape

Orca Mobile is a separate Expo and React Native app. It offers a read-mostly fleet view, recent output, chat or terminal views, prompt replies, notifications, and selected remote controls. Orca explicitly keeps the desktop host authoritative.[^orca-mobile]

Shepherd can use a simpler native path. `ShepherdCore`, `ShepherdProtocol`, and `ShepherdRemote` use Foundation, Dispatch, and Darwin, but the root Swift package currently declares only macOS. `RemoteHostStore` should not move to iOS because it mixes transport, Mac settings, themes, Ghostty models, and `UserDefaults` token storage.[^shepherd-package][^shepherd-remote-store] The pinned SwiftTerm dependency supports iOS 14 and includes a UIKit terminal view. The vendored Ghostty package declares iOS 15 support, but Shepherd's `TerminalSurfaceKit` adapter is AppKit-only.[^swiftterm-package][^swiftterm-ios][^shepherd-ghostty]

**Takeaway for Shepherd:** reuse the model and protocol modules, compose `RemoteHostClient` directly, store credentials in Keychain, and start with SwiftTerm's iOS view. That is the shortest proven route. Revisit Ghostty on iOS only if measured rendering needs justify a second adapter.

## Shepherd performance findings

These are code-path risks, not measured regressions. Phase 0 must establish baselines before changing behavior.

| ID | Finding | Likely impact | Confidence | First action |
| --- | --- | --- | --- | --- |
| F1 | Hidden mounted panes parse output in both SwiftTerm and Ghostty. | High CPU and memory as the fleet grows. | High | Measure visible versus hidden output, then cold-park old surfaces. |
| F2 | All PTY parsing, snapshots, remote work, and state mutations share one serial queue hierarchy. | A noisy PTY or large snapshot can delay unrelated sessions. | High | Instrument queue delay and snapshot time before changing queues. |
| F3 | Remote output creates one main-queue callback per decoded frame with no client-side byte bound. | UI backlog, heat, and battery drain, especially on mobile. | High | Add the same one-in-flight, bounded coalescer used by the local path. |
| F4 | Terminal output uses base64 NDJSON and per-viewer encoding. | About 33 percent wire expansion plus JSON and copy cost. | High | Add a negotiated binary stream after F3. |
| F5 | Any phone-sized remote viewport becomes the PTY size. | Desktop reflow, repeated TUI repaint, and poor passive monitoring. | High | Add observer and controller viewport roles. |
| F6 | Every state mutation validates, pretty-prints, atomically writes, and publishes the full state. | Queue stalls and fan-out cost as state grows. | High | Skip unchanged status writes, then separate runtime events from durable checkpoints. |
| F7 | Mounted terminal memory grows for the full app run. | Memory and SwiftUI hierarchy growth. | High | Measure resident memory per pane, then use the F1 parking policy. |

### F1. Dual parsing continues while hidden

`PTYSession` creates a `Data` chunk, `SessionScreen` converts it to `[UInt8]`, and SwiftTerm parses it. When the pane is locally attached, `TerminalSessionStore` forwards the same bytes to Ghostty on the main actor.[^shepherd-pty][^shepherd-screen][^shepherd-terminal-store] Hidden panes stop drawing, not parsing. Because agent layouts stay mounted, this cost scales with busy hidden agents rather than visible panes.

The host-side parser cannot simply be removed. It supplies exact snapshots, pane reads, remote attach, and recovery. The smallest useful change is to park old hidden view terminals while leaving the host model alive.

### F2. The serial queue is an ordering win and a fairness risk

Every session queue targets `shepherd.sessions`. `SessionScreen.feed` runs synchronously under that hierarchy. Snapshot serialization walks retained history and visible cells while holding the same ordering domain.[^shepherd-session-queue][^shepherd-screen] This makes the attach invariant easy to verify, but unrelated sessions can wait behind parsing and snapshot work.

Do not split queues first. Instrument time spent in PTY feed, snapshot creation, remote encoding, and state persistence. If queue wait becomes material, move screen parsing and per-session output bookkeeping to independent session queues. Keep workspace mutations and the session registry on one narrow authority queue. Attach then needs a per-session fence that captures snapshot sequence and registration atomically.

### F3. Remote clients need a delivery scheduler

`RemoteHostClient.handleReadable` drains available socket data, decodes each frame, and posts one closure to the main queue for each output frame. `RemotePaneSession.receive` feeds each closure directly to the terminal.[^shepherd-remote-client][^shepherd-remote-store] Socket backpressure cannot limit closures already posted to the UI thread.

Add one pending buffer and one in-flight callback per session inside `ShepherdRemote`. Use a leading edge: deliver immediately when idle, then coalesce the burst tail for a short measured window. Count pending bytes, not frames. On overflow, stop granting credit and request a fresh snapshot rather than growing memory.

### F4. Terminal bytes should leave NDJSON

Swift `Codable` encodes `Data` as base64. Shepherd chunks raw output at 256 KiB, copies each chunk, then JSON-encodes it separately for each viewer. Full-state broadcasts already encode once and reuse the payload, which shows the cheaper fan-out shape.[^shepherd-framing][^shepherd-remote-server]

Keep NDJSON for state and mutations. Add a negotiated binary frame containing stream ID, generation, opcode, sequence, and payload length. Encode each terminal chunk once per connection, multiplex sessions on that connection, and cap both per-stream and per-connection in-flight bytes.

### F5. Mobile must not own the viewport by accident

Shepherd currently ignores the local viewport whenever any remote viewer is present and applies the smallest remote grid. `PTYSession.resize` issues `SIGWINCH` even for an unchanged grid to force a repaint after reattachment.[^shepherd-viewport][^shepherd-resize] A phone attaching for observation can therefore reflow the terminal for everyone.

Add an attach role:

- `observe`: receives the host grid and output, never resizes the PTY;
- `claimViewport`: may report a grid and receives a lease or generation;
- `releaseViewport`: returns control to the prior claimant or host.

One controller owns the viewport. Other clients fit or scroll the received terminal. An explicit "control from this device" action can transfer the claim.

### F6. Runtime status should not force durable full-state work

`StateStore.update` copies and validates all state, pretty-prints sorted JSON, and atomically writes it before the server publishes a full snapshot. Agent status uses that path even when the reported status matches the current value.[^shepherd-state]

The first fix is one guard for unchanged status. Next, add a monotonic state revision and publish small runtime events such as agent status and child-run changes. Keep durable workspace mutations synchronous at first. If persistence is still measurable, checkpoint durable state with a bounded debounce and flush it on orderly shutdown. Do not weaken corrupt-state quarantine.

## Target architecture

The target keeps one authoritative Shepherd host and adds two real adapters at one seam: in-process macOS UI and remote transport. A later headless executable becomes a third adapter only where host operations differ.

```text
macOS SwiftUI app                          iPhone / iPad app
        │                                        │
        │ LocalHostAdapter                       │ RemoteHostAdapter
        ▼                                        ▼
              HostRuntime interface
        state stream · commands · terminal attachments
                           │
                           ▼
                 ShepherdHost runtime
     workspace authority · PTYs · screen models · persistence
                           │
             ┌─────────────┴─────────────┐
             │                           │
      control plane                terminal data plane
   structured messages          binary multiplexed frames
 state revisions/capabilities   seq · ACK · snapshot recovery
             │                           │
             └──────── direct TCP / VPN ┘
                           │
                  optional outbound relay
                  after direct mode ships
```

### HostRuntime interface

This seam should be small. Do not expose every current `SessionServer` method. The interface includes its ordering and error contracts, not just Swift signatures.

```swift
protocol HostRuntime: Sendable {
    func snapshot() async throws -> HostSnapshot
    func events(after revision: UInt64?) -> AsyncStream<HostEvent>
    func perform(_ command: HostCommand) async throws -> HostCommandResult
    func attach(_ request: TerminalAttachRequest) async throws -> TerminalAttachment
}

protocol TerminalAttachment: Sendable {
    var frames: AsyncStream<TerminalFrame> { get }
    func send(_ input: TerminalInput) async throws
    func acknowledge(through sequence: UInt64)
    func claimViewport(_ viewport: TerminalViewport) async throws -> ViewportLease
    func detach()
}
```

This is a design sketch, not an implementation prescription. Validate it against the first two adapters before committing it to source. One adapter would make the seam hypothetical. Local and remote adapters make it real.

`HostCommand` should cover the operations actually needed by both clients: create agent, create space, pane changes, agent rename or deletion, and automation run control. App-only actions such as window focus and sheets stay outside the interface.

### Control plane

Use structured messages for:

- authentication and device identity;
- host snapshot and state revisions;
- capability negotiation;
- workspace and pane commands;
- model and directory queries;
- terminal subscribe, detach, viewport claim, and recovery requests;
- typed errors.

Keep unknown optional fields ignorable. New behavior that an old peer cannot safely ignore requires a capability. Add cross-version tests before allowing a version range rather than exact equality.

### Terminal data plane

Use binary frames on a separate ordered connection or a framed substream. Each stream has:

- a compact stream ID;
- a stream generation that changes on reattach or replacement;
- an opcode;
- an absolute end-byte sequence;
- payload length and raw bytes.

Required opcodes are `snapshot`, `output`, `exit`, `ack`, `pause`, `resume`, `claimViewport`, and `releaseViewport`. New opcodes require negotiation and numbers never get reused.

Flow-control rules:

1. Host sends only within per-stream and connection credit.
2. Client acknowledges bytes after its terminal parser accepts them, not when the socket reads them.
3. Host pauses that stream when credit is exhausted. A noisy stream cannot consume every connection byte.
4. If retained live output crosses its cap, host discards the stale range and sends a new authoritative snapshot plus sequence.
5. Scheduling is round-robin across streams, with the currently visible stream first.
6. Disconnect drops view state. Reconnect requests a snapshot from the authoritative host model.

These rules preserve Shepherd's existing no-loss local path while giving remote clients a bounded recovery path.

### Host process and headless mode

The first host executable can embed `ShepherdHost` and retain app-scoped PTY lifetime. It must own all operations now routed through `ShepherdViewModel`, including pane authorization, agent creation, automation lifecycle, and settings needed to spawn agents. The macOS app becomes a local adapter and view, not a required host implementation.

Only after this works should Shepherd consider a separate PTY daemon. If added, supervise it separately from the host process and document exactly which restart modes preserve sessions. Orca's cgroup warning shows why `fork` plus `setsid` is not enough.[^orca-orcad]

### Mobile client

The initial iOS and iPadOS package graph should be:

```text
ShepherdCore ───────┐
                    ├── ShepherdRemote ── ShepherdMobileApp
ShepherdProtocol ───┘                         ├── SwiftUI
                                              ├── Keychain adapter
                                              └── SwiftTerm UIKit terminal
```

Build a mobile-specific connection store around `RemoteHostClient`. Keep it responsible for saved hosts, reconnect, selected session, notifications, and visible attachments. Store per-device credentials in Keychain. Attach only the visible terminal and perhaps one short warm predecessor. Do not import `ShepherdSessions`, `ShepherdApp`, `TerminalSurfaceKit`, Sparkle, or Mac settings.

First-release screens:

1. Hosts and connection health.
2. Spaces and agents with status.
3. Agent detail with recent output and a prompt composer.
4. Raw terminal with phone keyboard helpers.
5. New agent flow.

The existing remote protocol supports viewing, input, paste, pane control, directory and model listing, and creating spaces and agents. It does not yet support full remote parity such as agent rename or deletion, space deletion, automation management, or host settings.[^shepherd-protocol] Add only the commands the mobile product actually exposes.

## Migration plan

### Phase 0. Measure and pin invariants

**Goal:** know where time and memory go before changing architecture.

Add repeatable benchmarks or signposts for:

- keystroke to visible echo at idle and during a flood;
- main-thread terminal-feed duration and queued remote bytes;
- fairness between one flooding session and one interactive session;
- `SessionScreen.feed` throughput and snapshot time at 80x24, 200x60, and 2,000 history lines;
- resident memory for 1, 10, and 50 mounted panes;
- hidden-pane CPU while visible and hidden sessions emit fixed byte rates;
- whole-state persist and remote publication time at representative fleet sizes;
- attach and reconnect time, replay bytes, and duplicate or missing-byte checks.

Preserve these invariants in focused tests:

- snapshot plus live output is exact and ordered;
- exit arrives only after buffered output;
- a stalled renderer bounds memory and eventually receives every local byte;
- hidden-surface restoration does not duplicate output;
- a passive remote viewer does not resize the PTY.

**Exit:** baseline numbers are recorded in a checked-in Markdown result and the benchmark commands run locally and in CI where stable.

### Phase 1. Low-risk fixes

1. Skip persistence and full-state publication for unchanged agent status while retaining the status callback used by launch UI.
2. Avoid no-op effective viewport resizes except on the explicit repaint path.
3. Add a bounded, one-in-flight remote output coalescer in `ShepherdRemote`.
4. Add counters and logs for remote pending bytes, coalesced deliveries, overflows, snapshot time, and state-persist time.

**Exit:** existing tests pass, new focused tests cover each guard, and echo latency does not regress.

### Phase 2. Cold-park hidden surfaces

Keep the active layout and a small recent set mounted. After a measured hidden delay, detach and release older Ghostty surfaces but keep their PTYs and `SessionScreen` models. Restore with the existing atomic snapshot and watermark path.

Start with one policy, not user settings: 30 seconds hidden, four recently active agent layouts, and a five-minute recent-use window. Tune only from measurements. Inspectors and global shells may need separate caps because their usage differs.

**Exit:** a 50-pane workload shows lower memory and hidden CPU, while recent switching stays within the agreed latency budget and replay tests remain exact.

### Phase 3. Binary terminal stream and viewport claims

Add negotiated capabilities for binary terminal frames, ACK flow control, recovery snapshots, and viewport claims. Keep the NDJSON path as a compatibility adapter during rollout. New clients prefer binary, old clients keep working on version 1.

Add tests for:

- mixed old and new peers;
- ACK window exhaustion and resume;
- one flooding stream alongside one interactive stream;
- client suspension and reconnect;
- pending overflow replaced by one snapshot;
- stale ACK and stale viewport generation rejection;
- passive phone attachment while a desktop retains its grid.

**Exit:** binary transport cuts CPU or wire bytes in the benchmark, slow clients stay bounded, and cross-version tests pass in both directions.

### Phase 4. Incremental state stream

Add host revision numbers and typed state events. Full snapshot remains the recovery path. Separate ephemeral runtime updates from durable workspace checkpoints. Keep structural mutations validated before publication.

Do not introduce an event-sourced store. Shepherd needs a current snapshot plus deltas, not a permanent command log.

**Exit:** status storms no longer rewrite and rebroadcast the full state, reconnect from a stale revision recovers with either deltas or one snapshot, and corrupt-state quarantine remains unchanged.

### Phase 5. Transport-neutral host runtime

Move host operations currently implemented by `ShepherdViewModel` behind the `HostRuntime` seam. Build two adapters:

- local macOS adapter used by `ShepherdApp`;
- remote adapter used by macOS and mobile clients.

Then add a headless macOS host executable using the same runtime. It starts with foreground lifetime and explicit shutdown. Do not add a daemon manager in this phase.

**Exit:** the app and headless executable pass the same host contract tests, and remote agent or pane creation works with no SwiftUI view model present.

### Phase 6. Native mobile companion

Make Core, Protocol, and Remote build for iOS. Add the mobile app with Keychain credentials, reconnect, state list, visible-session attach, prompt composer, raw SwiftTerm terminal, and notifications.

Ship direct Tailscale or LAN connectivity first. The listener must not be advertised as safe on the public internet.

**Exit:** an iPhone can pair on a private path, watch agent status, read recent output, answer a prompt, reconnect after suspension, and create an agent without resizing the host terminal.

### Phase 7. Optional outbound relay

Add this only if direct-network setup blocks adoption. The host and phone both establish outbound encrypted connections. The relay authenticates endpoints, applies strict byte and connection budgets, and forwards opaque frames. Host state and terminal recovery remain authoritative on the Shepherd machine.

Before implementation, write separate threat-model and operations documents covering device enrollment, revocation, key rotation, end-to-end encryption, abuse limits, regional routing, logs, outages, and decommissioning.

**Exit:** direct mode and relay mode pass the same protocol suite. Relay loss never loses host state or PTYs.

### Phase 8. Optional durable PTY owner

Consider a separately supervised PTY process only if users need sessions to survive host upgrades or crashes. This phase requires stable session and process generations, adoption, health checks that spawn a real PTY, version adapters, crash-loop limits, log rotation, explicit decommissioning, and OS-specific supervision.

Do not call this complete until destructive supervisor restarts are tested. Process detachment alone is insufficient.

## What not to copy from Orca

- Electron and React Native. They add a second UI and packaging stack without improving Shepherd's native Mac experience.
- xterm on the host. Replacing SwiftTerm would risk replay fidelity for no demonstrated gain.
- a cloud relay in the first mobile release. Tailscale or LAN proves the product with far less operational work.
- the full Orca RPC breadth. Add only commands used by Shepherd clients.
- daemon extraction before a headless host. That combines two lifecycle changes and makes failures hard to attribute.
- per-setting tuning for batching, parking, and queue limits. Start with measured constants and one policy.
- a permanent event log for workspace state. Snapshot plus bounded deltas is enough.

## Risks and rollback

| ID | Risk | Control | Rollback |
| --- | --- | --- | --- |
| R1 | Output batching raises interactive latency. | Leading-edge immediate delivery, visible-stream priority, echo benchmark. | Disable negotiated batching capability. |
| R2 | Parking brings back blank frames or duplicate replay. | Reuse snapshot watermark, hysteresis, restoration tests. | Keep all local surfaces mounted. |
| R3 | Per-session queues weaken ordering. | Introduce sequence fences before moving work, test attach under flood. | Keep parsing on the server queue. |
| R4 | Binary protocol strands older clients. | Capability negotiation, dual transport, bidirectional version-skew tests. | Fall back to NDJSON. |
| R5 | Mobile resize changes desktop output. | Observer default and explicit viewport lease. | Reject all mobile resize frames. |
| R6 | Deferred persistence loses the last runtime update. | Defer only ephemeral state first, flush durable checkpoints on shutdown. | Return durable mutations to synchronous writes. |
| R7 | Headless extraction duplicates host logic. | One `HostRuntime` interface and shared contract tests. | Keep the app as the only host adapter. |
| R8 | Relay creates a public security and operations burden. | Separate approval gate after direct mode, E2EE, per-device revocation, strict budgets. | Keep direct mode as the supported path. |

## Recommended issue order

1. Benchmark and signpost terminal paths.
2. Skip unchanged status persistence and no-op viewport resize.
3. Bound and coalesce remote client delivery.
4. Cold-park hidden surfaces.
5. Add binary streaming, ACKs, recovery snapshots, and viewport claims.
6. Add state revisions and deltas.
7. Extract host operations from the view model and ship a headless Mac host.
8. Ship the direct-network iPhone and iPad companion.
9. Evaluate a relay from mobile usage data.
10. Evaluate durable PTY ownership from restart-survival demand.

The first six items improve the existing Mac product even if mobile work stops. That is the right dependency order.

## Sources

### Shepherd

[^shepherd-session-queue]: Shepherd, [`SessionServer.swift` lines 66-85 and 1603-1637](https://github.com/jhartzell/shepherd/blob/8282abde021814af6e29057aefa01c82d2c46c56/Sources/ShepherdSessions/SessionServer.swift#L66-L85), plus the [session creation path](https://github.com/jhartzell/shepherd/blob/8282abde021814af6e29057aefa01c82d2c46c56/Sources/ShepherdSessions/SessionServer.swift#L1603-L1637).

[^shepherd-output]: Shepherd, [`SessionServer.swift` atomic attach and output delivery](https://github.com/jhartzell/shepherd/blob/8282abde021814af6e29057aefa01c82d2c46c56/Sources/ShepherdSessions/SessionServer.swift#L1644-L1677) and [`deliverOutput` through `finishOutputDelivery`](https://github.com/jhartzell/shepherd/blob/8282abde021814af6e29057aefa01c82d2c46c56/Sources/ShepherdSessions/SessionServer.swift#L1787-L1864).

[^shepherd-output-tests]: Shepherd, [`OutputCoalescingTests.swift`](https://github.com/jhartzell/shepherd/blob/8282abde021814af6e29057aefa01c82d2c46c56/Tests/ShepherdSessionsTests/OutputCoalescingTests.swift#L6-L204).

[^shepherd-pty]: Shepherd, [`PTYSession.swift` bounded drain and screen feed](https://github.com/jhartzell/shepherd/blob/8282abde021814af6e29057aefa01c82d2c46c56/Sources/ShepherdSessions/PTYSession.swift#L377-L403).

[^shepherd-screen]: Shepherd, [`SessionScreen.swift` ownership, feed, and snapshot`](https://github.com/jhartzell/shepherd/blob/8282abde021814af6e29057aefa01c82d2c46c56/Sources/ShepherdSessions/SessionScreen.swift#L4-L99) and [cell serialization](https://github.com/jhartzell/shepherd/blob/8282abde021814af6e29057aefa01c82d2c46c56/Sources/ShepherdSessions/SessionScreen.swift#L101-L178).

[^shepherd-terminal-store]: Shepherd, [`TerminalSessions.swift` output forwarding](https://github.com/jhartzell/shepherd/blob/8282abde021814af6e29057aefa01c82d2c46c56/Sources/ShepherdApp/TerminalSessions.swift#L115-L150) and [server callback wiring](https://github.com/jhartzell/shepherd/blob/8282abde021814af6e29057aefa01c82d2c46c56/Sources/ShepherdApp/TerminalSessions.swift#L238-L283).

[^shepherd-mounting]: Shepherd, [`WorkspaceSelection.swift`](https://github.com/jhartzell/shepherd/blob/8282abde021814af6e29057aefa01c82d2c46c56/Sources/ShepherdApp/WorkspaceSelection.swift#L4-L67), [`WorkspaceView.swift`](https://github.com/jhartzell/shepherd/blob/8282abde021814af6e29057aefa01c82d2c46c56/Sources/ShepherdApp/WorkspaceView.swift#L14-L45), and [`TerminalSurfaceModel.swift` render occlusion](https://github.com/jhartzell/shepherd/blob/8282abde021814af6e29057aefa01c82d2c46c56/Sources/TerminalSurfaceKit/TerminalSurfaceModel.swift#L231-L273).

[^shepherd-remote-server]: Shepherd, [`SessionServer.swift` remote attach and output encoding](https://github.com/jhartzell/shepherd/blob/8282abde021814af6e29057aefa01c82d2c46c56/Sources/ShepherdSessions/SessionServer.swift#L764-L856) and [bounded connection writes](https://github.com/jhartzell/shepherd/blob/8282abde021814af6e29057aefa01c82d2c46c56/Sources/ShepherdSessions/SessionServer.swift#L1125-L1189).

[^shepherd-remote-client]: Shepherd, [`RemoteHostClient.swift` socket drain and main-queue callbacks](https://github.com/jhartzell/shepherd/blob/8282abde021814af6e29057aefa01c82d2c46c56/Sources/ShepherdRemote/RemoteHostClient.swift#L448-L533).

[^shepherd-framing]: Shepherd, [`Framing.swift`](https://github.com/jhartzell/shepherd/blob/8282abde021814af6e29057aefa01c82d2c46c56/Sources/ShepherdProtocol/Framing.swift#L3-L60).

[^shepherd-protocol]: Shepherd, [`RemoteMessage.swift` protocol, capabilities, and requests](https://github.com/jhartzell/shepherd/blob/8282abde021814af6e29057aefa01c82d2c46c56/Sources/ShepherdProtocol/RemoteMessage.swift#L4-L96) and [host replies](https://github.com/jhartzell/shepherd/blob/8282abde021814af6e29057aefa01c82d2c46c56/Sources/ShepherdProtocol/RemoteMessage.swift#L261-L298).

[^shepherd-handshake]: Shepherd, [`SessionServer.swift` exact version check and token authentication](https://github.com/jhartzell/shepherd/blob/8282abde021814af6e29057aefa01c82d2c46c56/Sources/ShepherdSessions/SessionServer.swift#L500-L535).

[^shepherd-viewport]: Shepherd, [`SessionServer.swift` smallest-viewer viewport policy](https://github.com/jhartzell/shepherd/blob/8282abde021814af6e29057aefa01c82d2c46c56/Sources/ShepherdSessions/SessionServer.swift#L736-L803) and [`RemoteSessionStreamTests.swift` phone-sized example](https://github.com/jhartzell/shepherd/blob/8282abde021814af6e29057aefa01c82d2c46c56/Tests/ShepherdSessionsTests/RemoteSessionStreamTests.swift#L239-L280).

[^shepherd-resize]: Shepherd, [`PTYSession.swift` resize and same-size repaint`](https://github.com/jhartzell/shepherd/blob/8282abde021814af6e29057aefa01c82d2c46c56/Sources/ShepherdSessions/PTYSession.swift#L273-L289).

[^shepherd-state]: Shepherd, [`StateStore.swift` full validation and atomic persistence](https://github.com/jhartzell/shepherd/blob/8282abde021814af6e29057aefa01c82d2c46c56/Sources/ShepherdSessions/StateStore.swift#L31-L41) and [JSON write](https://github.com/jhartzell/shepherd/blob/8282abde021814af6e29057aefa01c82d2c46c56/Sources/ShepherdSessions/StateStore.swift#L83-L95), plus [`SessionServer.swift` status publication](https://github.com/jhartzell/shepherd/blob/8282abde021814af6e29057aefa01c82d2c46c56/Sources/ShepherdSessions/SessionServer.swift#L1212-L1234).

[^shepherd-package]: Shepherd, [`Package.swift`](https://github.com/jhartzell/shepherd/blob/8282abde021814af6e29057aefa01c82d2c46c56/Package.swift#L4-L68).

[^shepherd-remote-store]: Shepherd, [`RemoteHostStore.swift` host config and mixed responsibilities](https://github.com/jhartzell/shepherd/blob/8282abde021814af6e29057aefa01c82d2c46c56/Sources/ShepherdApp/RemoteHostStore.swift#L1-L80), [transport callback bridge](https://github.com/jhartzell/shepherd/blob/8282abde021814af6e29057aefa01c82d2c46c56/Sources/ShepherdApp/RemoteHostStore.swift#L181-L210), and [Mac terminal adapter](https://github.com/jhartzell/shepherd/blob/8282abde021814af6e29057aefa01c82d2c46c56/Sources/ShepherdApp/RemoteHostStore.swift#L347-L481).

[^shepherd-ghostty]: Shepherd, [vendored Ghostty package platform declarations](https://github.com/jhartzell/shepherd/blob/8282abde021814af6e29057aefa01c82d2c46c56/Vendor/libghostty-spm/Package.swift#L4-L14) and [`TerminalSurfaceModel.swift` AppKit dependency](https://github.com/jhartzell/shepherd/blob/8282abde021814af6e29057aefa01c82d2c46c56/Sources/TerminalSurfaceKit/TerminalSurfaceModel.swift#L1-L8).

### Orca

[^orca-orcad]: Orca, [`docs/reference/orcad-operations.md`](https://github.com/stablyai/orca/blob/67e22345daf882190911355eed152ef33e051c5e/docs/reference/orcad-operations.md#two-long-lived-processes-not-one), including the warning about systemd cgroup shutdown.

[^orca-output-plane]: Orca, [`session-output-plane.ts`](https://github.com/stablyai/orca/blob/67e22345daf882190911355eed152ef33e051c5e/src/main/daemon/session-output-plane.ts#L30-L244) and [`session-output-pipeline.ts`](https://github.com/stablyai/orca/blob/67e22345daf882190911355eed152ef33e051c5e/src/main/daemon/session-output-pipeline.ts#L1-L31).

[^orca-daemon-batcher]: Orca, [`daemon-stream-data-batcher.ts`](https://github.com/stablyai/orca/blob/67e22345daf882190911355eed152ef33e051c5e/src/main/daemon/daemon-stream-data-batcher.ts), including `STREAM_DATA_BATCH_INTERVAL_MS = 2` and the 128 KiB shallow socket gate.

[^orca-producer-pause]: Orca, [`session-producer-pause.ts`](https://github.com/stablyai/orca/blob/67e22345daf882190911355eed152ef33e051c5e/src/main/daemon/session-producer-pause.ts#L1-L38).

[^orca-flow-control]: Orca, [`terminal-multiplex-flow-control.ts`](https://github.com/stablyai/orca/blob/67e22345daf882190911355eed152ef33e051c5e/src/main/runtime/rpc/methods/terminal/terminal-multiplex-flow-control.ts) and [`terminal-stream-protocol.ts`](https://github.com/stablyai/orca/blob/67e22345daf882190911355eed152ef33e051c5e/src/shared/terminal-stream-protocol.ts).

[^orca-renderer-drain]: Orca, [`pane-terminal-output-drain.ts`](https://github.com/stablyai/orca/blob/67e22345daf882190911355eed152ef33e051c5e/src/renderer/src/lib/pane-manager/pane-terminal-output-drain.ts) and [`pane-terminal-output-scheduler.ts`](https://github.com/stablyai/orca/blob/67e22345daf882190911355eed152ef33e051c5e/src/renderer/src/lib/pane-manager/pane-terminal-output-scheduler.ts).

[^orca-mobile-coalescer]: Orca, [`terminal-write-coalescer.ts`](https://github.com/stablyai/orca/blob/67e22345daf882190911355eed152ef33e051c5e/mobile/src/terminal/terminal-write-coalescer.ts#L1-L67).

[^orca-parking]: Orca, [`terminal-hidden-view-parking.ts`](https://github.com/stablyai/orca/blob/67e22345daf882190911355eed152ef33e051c5e/src/renderer/src/components/terminal-pane/terminal-hidden-view-parking.ts), including its 30-second delay, five-minute retention window, and recent-set caps.

[^orca-wire]: Orca, [`docs/reference/remote-wire-compatibility.md`](https://github.com/stablyai/orca/blob/67e22345daf882190911355eed152ef33e051c5e/docs/reference/remote-wire-compatibility.md).

[^orca-relay]: Orca, [`cloud/README.md`](https://github.com/stablyai/orca/blob/67e22345daf882190911355eed152ef33e051c5e/cloud/README.md#orca-relay).

[^orca-mobile]: Orca, [`docs/site/content/docs/mobile.mdx`](https://github.com/stablyai/orca/blob/67e22345daf882190911355eed152ef33e051c5e/docs/site/content/docs/mobile.mdx) and [`mobile/README.md`](https://github.com/stablyai/orca/blob/67e22345daf882190911355eed152ef33e051c5e/mobile/README.md).

### Mobile terminal dependencies

[^swiftterm-package]: SwiftTerm 1.18.0, [`Package.swift` iOS 14 declaration](https://github.com/migueldeicaza/SwiftTerm/blob/7691f85b222a67a66b58499e1b2647443cf0dda7/Package.swift#L120-L124), pinned by Shepherd's [`Package.resolved`](https://github.com/jhartzell/shepherd/blob/8282abde021814af6e29057aefa01c82d2c46c56/Package.resolved#L32-L38).

[^swiftterm-ios]: SwiftTerm 1.18.0, [`README.md` UIKit terminal view](https://github.com/migueldeicaza/SwiftTerm/blob/7691f85b222a67a66b58499e1b2647443cf0dda7/README.md#ios-uiview).
