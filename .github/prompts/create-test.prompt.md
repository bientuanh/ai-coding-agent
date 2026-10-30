---
description: Create unit tests following the project's Jest conventions with coverage above 80%
---

Use the `create-unitest` skill.

Create or update unit tests for the code described by the user.

Before implementation:

1. Understand the expected behavior and main test cases.
2. Read the target source file and its nearby dependencies.
3. Read the existing tests and Jest configuration.
4. Follow the closest existing test style.
5. Mock databases, external APIs, Firebase, and other integrations when testing units.

Test the success, failure, boundary, and error-propagation cases that matter for the target behavior.

After implementation:

1. Run the focused test file.
2. Run the full Jest suite with coverage.
3. Confirm global branches, functions, lines, and statements coverage is above 80%.
4. Run type checking and linting when the change affects TypeScript source or tests.
5. Review the final diff and do not commit generated coverage output.
