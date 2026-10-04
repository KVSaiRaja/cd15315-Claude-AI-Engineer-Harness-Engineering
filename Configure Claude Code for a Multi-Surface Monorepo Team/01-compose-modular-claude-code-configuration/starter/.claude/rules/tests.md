---
description: Conventions for co-located TypeScript test files
paths:
  - "**/*.test.tsx"
  - "**/*.test.ts"
---

# Test file rules

- Use explicit assertions for behavior and outcomes.
- Do not add snapshot-only tests for new functionality.
- Keep test data deterministic and easy to understand.
- Avoid real database calls in unit tests.
- Avoid real network requests in unit tests.
- Do not use arbitrary `setTimeout` delays to synchronize tests.
- Prefer focused test cases that verify one behavior at a time.
- Reuse established test-data builders when they exist.
- Keep tests co-located with the source file they exercise.
- Test both successful behavior and relevant error behavior.
- Preserve the existing assertion style when modifying an established test suite.
- Avoid mocking internal implementation details when testing observable behavior.
