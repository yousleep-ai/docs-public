# youSleep developer documentation

The public developer site for the youSleep Portal: the Python SDK, the REST
API, and how to package an analysis to run on the platform.

This repository holds no documentation of its own beyond the site's spine
(`docs/`): the pages come from the repositories of the things they describe,
and `sources.yml` lists exactly which of them are published here. The site is
built by the private documentation toolchain (`yousleep-ai/docs`) through its
reusable workflow, so the internal and the public site share one theme, one
set of generators and one set of rules.

- **Publishing a page** is listing it in `sources.yml`. That makes a change to
  this file a publication decision, reviewed as one.
- **Editing a page** happens in the repository that owns it.
- **The spine** — the home page, *Start here*, *Registration* — lives in `docs/`
  here, because it describes how the parts fit rather than any one part.
