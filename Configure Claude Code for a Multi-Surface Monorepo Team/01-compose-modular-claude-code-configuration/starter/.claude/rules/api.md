---
description: Conventions for API handler files
paths:
  - "src/api/**/*"
---

# API handler rules

- Keep API handlers thin and focused on HTTP concerns.
- Validate request input before calling application logic.
- Use the repository or service layer for business operations.
- Do not place SQL queries directly in files under `src/api/`.
- Return consistent HTTP status codes for successful operations.
- Return structured error responses for expected client errors.
- Do not expose database implementation details in API responses.
- Keep authentication and authorization checks explicit in each protected handler.
- Use the shared API error type for expected domain and validation failures.
- Preserve the handler's established request and response shape when modifying existing endpoints.
- Keep external service calls behind the appropriate service abstraction.
- Add or update a focused test when changing an API handler's behavior.
