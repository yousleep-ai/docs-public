# youSleep developer documentation

The public developer site for the youSleep Portal, published at
[docs.yousleep.ai](https://docs.yousleep.ai/).
It covers the Python SDK, the REST API and how to package an analysis to run
on the platform.

## How the site is organised

- `docs/` holds the pages that describe how the parts fit together: the home
  page, *Start here*, *Registration* and the YASA packaging example.
- Every other page is maintained in the repository of the component it
  describes and is pulled in at build time. `sources.yml` lists which pages
  are published. Adding a page to that list is a publication decision and is
  reviewed as one.
- The site is built by the youSleep documentation toolchain, which the
  workflow in `.github/workflows/` checks out, so the site shares its theme
  and build rules with the rest of the youSleep documentation. The build
  fails if the output contains anything beyond the listed pages or text that
  is not meant to be public, so a pull request here is checked the same way
  before it can merge.

## Contributing

To report an error on a page, open an issue here. To propose a change to a
page under `components/`, open a pull request in the repository that owns it;
for the Python SDK that is [`yousleep-ai/common`](https://pypi.org/project/yousleep-common/)
on PyPI. Questions go to [contact@yousleep.ai](mailto:contact@yousleep.ai).
