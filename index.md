# Towards immutable Butler collections

```{abstract}
Data release processing is spread across multiple sites.  Pipeline  workflows require data to be organized in Butler collections, which must be replicated at each site.

Butler collections are mutable, which creates challenges for reliably synchronizing them between sites and ensuring that the correct set of data is present for pipeline workflows.

This tech note proposes a tool for creating pseudo-immutable, point-in-time snapshots of Butler collections.  This would be useful for solving immediate data movement problems, and is a building block towards future work to improve the reliability and performance of data synchronization across sites.
```

## Add content here

See the [Documenteer documentation](https://documenteer.lsst.io/technotes/index.html) for tips on how to write and configure your new technote.
