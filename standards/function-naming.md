# Function Naming

## Purpose

Names should reveal caller-relevant intent, not transport plumbing or incidental implementation detail.

## Core Rule

Function names expose only the concerns the caller needs to know.

Generated, transport, framework, and vendor names may exist at their boundary, but should not leak into application-facing domain, hook, or UI names unless the caller genuinely needs that detail.

## Layer Guidance

- Generated or vendor code: keep generated names unchanged.
- Adapter/API boundary: transport terms are acceptable when the function directly wraps transport.
- Domain/query/mutation factories: use business intent and cache ownership.
- React hooks: name the state or server interaction the caller consumes.
- UI handlers: name the user interaction or local transition.
- Mappers: prefer `map`, `to`, `from`, or policy-specific verbs.

## Avoid

- Names based only on HTTP method, endpoint version, controller names, or batching mechanics.
- Names that hide important domain policy.
- Names that imply a broader behavior than the function actually performs.

## Review Checklist

- Does the public name describe caller intent?
- Is generated/vendor naming contained at the boundary?
- Would a caller know when to use this without reading implementation?
- Are similar operations named consistently across the same layer?
