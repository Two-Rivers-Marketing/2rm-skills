# CONTEXT — Time logging & project registry

Glossary for the `log-time` skill and the project registry it reads. Terms only — no
implementation. Job numbers, examples, and field contracts live in
[REFERENCE.md](REFERENCE.md).

## Glossary

- **Job** — a Workamajig billing unit, identified by a job number (e.g. `WG-26503`). The thing
  time is logged *against*. Owned by Workamajig; the registry only caches it. (Workamajig calls
  this a "project"; we never use that word for it, to avoid collision with Workspace.)

- **Client** — the Workamajig account a set of Jobs belongs to (e.g. Western Global). Groups Jobs
  in the registry.

- **Workspace** — a local directory where operator works. Contains **one or more Jobs**. A Workspace
  is *not* 1:1 with a Job — e.g. one `western-global/` folder spans `WG-26503`, `WG-26504`,
  `WG-25505`. "Directory awareness" means resolving the current Workspace to its *candidate* Jobs,
  then narrowing — never assuming one.

- **Registry** — the local file that is a refreshable **cache of Workamajig** (Clients + Jobs) plus
  a **local Overlay**. Merged on refresh by job number: refresh overwrites WMJ-owned fields, never
  Overlay fields.

- **Overlay** — the local-only data Workamajig cannot hold: Workspace paths, aliases, and
  per-Job default task/service. Hand-owned by the user; preserved across refreshes.

- **Refresh** — re-seeding the Workamajig cache (`clients`) from a recent window of the user's time
  entries. Overwrites the cache; never touches the Overlay.

- **Self-heal** — when a referenced Job isn't in the cache, resolving it live from Workamajig and
  appending it to the cache. The reason a stale cache never blocks logging.
