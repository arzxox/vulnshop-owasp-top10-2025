# Security Policy

This repository is **intentionally vulnerable software** built for
security education (OWASP Top 10:2025 training). It contains deliberate
SQL injection, XSS, command injection, insecure deserialization, and other
flaws described in the README.

- Do **not** deploy this application on any publicly reachable host.
- Do **not** enter real credentials, personal data, or anything sensitive
  into it.
- Run it only on `localhost` or in an isolated VM/container/network with
  no other reachable services.
- There is no supported way to "report a vulnerability" in this app —
  finding and exploiting them is the entire point. If you find a bug in
  the *training material itself* (e.g. a walkthrough that no longer
  works, or an exploit that doesn't reproduce), please open a GitHub
  issue instead.

By using this repository you accept full responsibility for how and where
you run it.
