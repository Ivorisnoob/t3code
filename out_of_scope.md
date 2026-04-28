# Out of Scope / Pending Verification

- The exact payload of Shiki syntax highlighting XSS bypassing needs runtime testing to prove exploitability.
- The `node:sqlite` database queries use Effect SQL which seems safe (parameterized), but complex dynamic queries should be further audited with a fuzzer.
