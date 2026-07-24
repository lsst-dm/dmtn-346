[![Website](https://img.shields.io/badge/dmtn--346-lsst.io-brightgreen.svg)](https://dmtn-346.lsst.io)
[![CI](https://github.com/lsst-dm/dmtn-346/actions/workflows/ci.yaml/badge.svg)](https://github.com/lsst-dm/dmtn-346/actions/workflows/ci.yaml)

# Towards immutable Butler collections

## DMTN-346

Data release processing is spread across multiple sites.  Pipeline  workflows require data to be organized in Butler collections, which must be replicated at each site.

Butler collections are mutable, which creates challenges for reliably synchronizing them between sites and ensuring that the correct set of data is present for pipeline workflows.

This tech note proposes a tool for creating pseudo-immutable, point-in-time snapshots of Butler collections.  This would be useful for solving immediate data movement problems, and is a building block towards future work to improve the reliability and performance of data synchronization across sites.

**Links:**

- Publication URL: https://dmtn-346.lsst.io
- Alternative editions: https://dmtn-346.lsst.io/v
- GitHub repository: https://github.com/lsst-dm/dmtn-346
- Build system: https://github.com/lsst-dm/dmtn-346/actions/


## Build this technical note

You can clone this repository and build the technote locally if your system has Python 3.11 or later:

```sh
git clone https://github.com/lsst-dm/dmtn-346
cd dmtn-346
make init
make html
```

Repeat the `make html` command to rebuild the technote after making changes.
If you need to delete any intermediate files for a clean build, run `make clean`.

The built technote is located at `_build/html/index.html`.

## Publishing changes to the web

This technote is published to https://dmtn-346.lsst.io whenever you push changes to the `main` branch on GitHub.
When you push changes to a another branch, a preview of the technote is published to https://dmtn-346.lsst.io/v.

## Editing this technical note

The main content of this technote is in `index.md` (a Markdown file parsed as [CommonMark/MyST](https://myst-parser.readthedocs.io/en/latest/index.html)).
Metadata and configuration is in the `technote.toml` file.
For guidance on creating content and information about specifying metadata and configuration, see the Documenteer documentation: https://documenteer.lsst.io/technotes.
