# docs-public

The public developer site for the youSleep Portal. This repository is a rendering, not a source: `sources.yml` lists which pages of which repositories are published, `mkdocs.yml` names and arranges them, and `docs/` holds the spine (home, orientation, registration). The site is built by the youSleep documentation toolchain, which the workflow checks out.

Listing a page in `sources.yml` publishes it. Treat a change to that file as a publication decision, and edit a page in the repository that owns it. This repository is public: nothing in it, including comments and commit messages, should reference internal systems, issues or documents.
