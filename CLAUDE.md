# docs-public

The public developer site for the youSleep Portal. This repository is a rendering, not a source: `sources.yml` lists which pages of which repositories are published, `mkdocs.yml` names and arranges them, and `docs/` holds only the spine (home, orientation, registration). The build is another repository's toolchain, checked out by the workflow here, so this site and the internal one share one theme and one set of rules.

Listing a page in `sources.yml` publishes it. Treat a change to that file as a publication decision, and edit a page in the repository that owns it.
