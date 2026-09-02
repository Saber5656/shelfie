# Review resolution addendum

- PR: #1
- Base: `main`
- Resolution scope: the five existing review threads listed below.
- This document is a normative design addendum. It records the accepted resolution contract; it does not claim that implementation or tests have already run.
- Bot review is not retriggered.

## PRRT_kwDOTNkIr86OZ9oo — privacy-safe book lookup default

Problem: an unconditional Google fallback conflicts with the stated openBD-only privacy default.

Resolution: the default resolver MUST use openBD only. Google Books lookup MUST be an explicit user-visible opt-in/consent setting, with the setting name shown before any request and no silent fallback. The resolver and privacy documentation MUST distinguish the two modes.

Focused verification before resolving this thread: run default mode with openBD failure and assert no Google request; enable the explicit option and assert only then that the Google request path is used.

## PRRT_kwDOTNkIr86OZ9op — Vite/PWA base path

Problem: deployment under `/shelfie/` can break assets and PWA scope when paths assume root.

Resolution: production configuration MUST set Vite `base: '/shelfie/'`; manifest `scope` and `start_url`, service-worker registration, and asset URLs MUST remain under that prefix.

Focused verification before resolving this thread: build and inspect emitted assets, manifest, and service-worker scope for the exact prefix.

## PRRT_kwDOTNkIr86OZ9oq — service-worker privacy enforcement

Problem: a Pages meta CSP does not constrain all service-worker fetch behavior.

Resolution: worker privacy MUST be enforced in the worker code and/or a CI audit of the worker bundle, using a static allowlist of permitted fetch destinations and explicitly excluding address-data or other sensitive-data requests. A host-header requirement may be documented as a deployment gate, but the product MUST NOT claim that the page meta CSP alone protects the service worker.

Focused verification before resolving this thread: inspect worker fetch code and built worker bundle, run the CI audit, and exercise permitted and forbidden destinations.

## PRRT_kwDOTNkIr86OZ9ot — terminal resolution/retry state

Problem: permanently missing books can retry forever without a terminal state.

Resolution: lookup state MUST include terminal `not_found` or `manual_needed`, attempt count, retry/backoff state such as `nextRetryAt`, and a maximum attempt policy. Terminal items MUST not be automatically rescheduled; retry after terminal state MUST be explicit.

Focused verification before resolving this thread: simulate permanent misses and transient errors, then assert terminal transitions, bounded retries, and explicit manual retry.

## PRRT_kwDOTNkIr86OZ9ou — HTTPS cover URL allowlist

Problem: cover URLs may be returned as HTTP or from an unexpected host.

Resolution: normalize `http://books.google.com` to HTTPS only when the host/path is on the documented allowlist. Reject other schemes, hosts, and ambiguous URLs; the app MUST not fetch an arbitrary returned URL.

Focused verification before resolving this thread: test allowed HTTP normalization, allowed HTTPS, disallowed hosts/schemes, and malformed input.

## Verification status

The checks above are required acceptance criteria for implementation. This addendum intentionally reports no test result and no implementation-complete status.