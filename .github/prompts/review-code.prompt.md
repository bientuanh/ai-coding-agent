---
description: Review Express and Node.js code for bugs, security risks, regressions, and missing tests
---

Use the `review-code` skill.

Review the code described by the user:

$ARGUMENTS

Act as a senior Express and Node.js developer. Base the review on `docs/architecture/coding-conventions.md` and the applicable project instructions.

Before reporting:

1. Identify the review scope from the user request, changed files, or current diff.
2. Read the complete behavior path around the change: routes, middleware, validators, controllers, services, DAOs, models, types, and tests as relevant.
3. Trace success, validation failure, authentication failure, missing-resource, conflict, and unexpected-error paths.
4. Check for security, data integrity, response-contract, architectural, and regression risks.
5. Check whether focused tests cover the changed behavior and important branches.

Do not modify files unless the user explicitly asks for fixes. Do not report subjective style preferences or hypothetical issues without a concrete execution path.

Return the review in this order:

1. Findings first, ordered by severity: `Blocker`, `High`, `Medium`, then `Low`.
2. For each finding, include the issue title, severity, clickable file link with a line reference, impact, and remediation direction.
3. List open questions or assumptions.
4. Add a brief change summary.
5. If there are no findings, state that clearly and mention remaining test gaps or residual risk.

Use this finding format:

```text
### [High] Short issue title
[src/path/file.ts](src/path/file.ts#L42)

Explain the concrete failure, affected behavior, and recommended direction.
```

Only claim tests or commands passed when they were actually run. If validation is relevant, run the narrowest available check and report its result.
