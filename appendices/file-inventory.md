# Appendix A — File Inventory

This appendix expands the cluster table from `audit-output/cluster-inventory.md` with a per-language breakdown.

The source codebase is a single legacy application — **Bugzilla**, the open-source Perl-based bug-tracking system. The `discovery/` directory was imported as a single bulk snapshot on 2026-05-05. The cluster inventory lists 61 key source files (the primary modules, CGI entry points, and scripts identified during discovery). The full codebase comprises ~250 source files totalling ~111 kLOC across `.pm`, `.cgi`, and `.pl` extensions (see `audit-output/cluster-inventory.md` overview).

## Language Breakdown

Language totals derived from the 61 key files catalogued in the cluster inventory. File extensions are mapped to languages per the extension → language map in `methodology.md`.

| Language | Files | LOC | % |
|---|---|---|---|
| Perl (`.pm`) | 42 | 28 435 | 77.3% |
| Perl (`.cgi`) | 17 | 7 616 | 20.7% |
| Perl (`.pl`) | 2 | 749 | 2.0% |
| **Total** | **61** | **36 800** | **100%** |

> **Note**: All three extensions (`.pm`, `.cgi`, `.pl`) are Perl. The full codebase (~250 files, ~111 kLOC) is monolingual Perl — there are no JavaScript, Python, or other runtime source files in `discovery/bugzilla/`. Python appears only in `docs/requirements.txt` for the Sphinx documentation build toolchain (not shipped code).

## Per-Cluster Summary

| Cluster | Files | LOC | Largest Module |
|---|---|---|---|
| service-bug | 9 | 8 374 | `Bugzilla/Bug.pm` (5 123 LOC) |
| service-user | 11 | 8 298 | `Bugzilla/User.pm` (3 407 LOC) |
| service-product | 10 | 4 312 | `Bugzilla/Product.pm` (1 185 LOC) |
| service-attachment | 6 | 4 921 | `Bugzilla/Flag.pm` (1 309 LOC) |
| service-comment | 2 | 724 | `Bugzilla/Comment.pm` (646 LOC) |
| service-search | 13 | 8 160 | `Bugzilla/Search.pm` (3 561 LOC) |
| service-notification | 10 | 2 011 | `whine.pl` (673 LOC) |
| **Total** | **61** | **36 800** | — |

## Source

Reuses the file table from `audit-output/cluster-inventory.md`; per-language totals derived by mapping file extensions to languages and summing LOC. See `methodology.md` for the extension → language map used.
