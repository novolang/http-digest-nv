# Changelog

All notable changes to http-digest-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## [0.1.0] — 2026-11-23

### Added

- `digchallenge`, a reader of a `WWW-Authenticate: Digest` challenge, and `best_of`, which chooses the strongest of several.
- `digauth`, the `Authorization: Digest` header and its `response` value for MD5, MD5-sess, SHA-256 and SHA-256-sess, `qop=auth`, `qop=auth-int`, the form of RFC 2069 and a hashed user name.
- `digerr`, the faults of both.
