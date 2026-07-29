# Towards immutable Butler collections

```{abstract}
Data release processing is spread across multiple sites.  Pipeline workflows
require data to be organized in Butler collections, which must be replicated at
each site.

Butler collections are mutable, which creates challenges for reliably
synchronizing them between sites and ensuring that the correct set of data is
present for pipeline workflows.

There is also the issue of multiple campaigns running at the same time.  For
example, test campaigns for DR1 might be going while there is a need to re-run
older processing.  Having point-in-time snapshots allows you to have multiple
versions of a collections present in a repository at the same time.

This tech note proposes a tool for creating pseudo-immutable, point-in-time
snapshots of Butler collections.  This would be useful for solving immediate
data movement problems, and is a building block towards future work to improve
the reliability and performance of data synchronization across sites.
```

## User workflow

The general idea is that users can maintain mutable collections like
`LSSTCam/calib` using their existing tools, then take a point-in-time
"snapshot" of the collection when it is time to distribute it to other
repositories or sites.  The full structure of the collection is serialized
to a file that can be copied easily between sites.

This tooling also allows verification that the snapshotted collections
are complete and correct before use.

The basic user interface flow would look like:
```
# Create a snapshot collection named 'LSSTCam/calib/2026-07-28' from the
# input collection 'LSSTCam/calib' in the current repo, and write the data
# needed to re-create it to a JSON file.
$ butler collection-snapshot create LSSTCam/calib LSSTCam/calib/2026-07-28
Snapshot data written to LSSTCam_calib_2026_07_28.collection.jsonl

# Copy the JSON file to another site using whatever means is convenient.  It is
# assumed that the datasets in the collections are transferred using other
# means (e.g. Rucio, currently).
$ scp LSSTCam/calib/2026_07_28.collection.jsonl france:

# If needed, the JSON file can be used to provide a list of datasets that
# should be transferred.  (e.g. as an input to rucio-register or butler
# transfer-datasets).
$ butler collection-snapshot list-datasets LSSTCam_calib_2026_07_28.collection.jsonl
1118611a-334e-4d81-8f02-e9d7f244955f
50fc7ed3-6c2a-47d3-bab2-3b552e0b4b59
...


# Load the collection structure. This generates all the sub-collections of the
# chained collection, then the top level collection.  If any of the collections
# exist already, it verifies that they match the input.
# If any files are missing, the tool reports a list of dataset UUIDs
# so that they can be transferred.
$ butler collection-snapshot load LSSTCam_calib_2026_07_28.collection.jsonl
Creating SNAPSHOT/LSSTCam/calib/DM-53546/skyFrames_epoch00/sky-u.20260115a/375dc4de-bbfd-4fc0-b9fa-30d45fe9dc32
Creating SNAPSHOT/LSSTCam/calib/DM-53546/e7ab8157-222c-4489-98d7-cec4fde1ac29
Creating SNAPSHOT/LSSTCam/calib/16f2aa3b-7f72-4b67-beb4-48a471dc8c3a
...
Creating LSSTCam/calib/2026-07-28

or:

The following datasets are missing for SNAPSHOT/LSSTCam/calib/DM-53546/skyFrames_epoch00/sky-u.20260115a/375dc4de-bbfd-4fc0-b9fa-30d45fe9dc32:
1118611a-334e-4d81-8f02-e9d7f244955f
50fc7ed3-6c2a-47d3-bab2-3b552e0b4b59
...

# It is possible to verify the integrity of the underlying snapshots
# without needing the JSON file, using the hash embedded in the collection
# names.  This could be integrated into the DRP workflow to make sure that
# input chains have not been corrupted (e.g. by deletion of datasets contained
# in them.)
$ butler collection-snapshot verify LSSTCam/calib/2026-07-28
Collection structure is intact.
```

One change to existing practice is that campaign DRP input chains would no
longer reference a mutable collection like `LSSTCam/calib` -- instead, they
would point at a specific version like `LSSTCam/calib/2026-07-28`.  This gives
better reproducibility, and allows other users to transfer newer versions of
the collection without needing to coordinate with other campaigns.

The main benefit of this is that collections can be registered at other sites
without worrying about "time" at all.  Even if the collection or its children
has changed, it isn't harmful to re-register an old version, since it is
totally separate from the newer version.  This will make it easier for transfer
tooling like Rucio to re-create collection chains.

The top level collection `LSSTCam/calib` could still exist as a development
convenience.  For repositories outside the "source of truth" repository for a
given collection, an alias could be set up using a chained collection with a
single child, e.g. `LSSTCam/calib -> [LSSTCam/calib/2026-07-28]`.

## Snapshot collection structure

Chained collections can be thought of as a tree: each chained collection adds
more branches, with run/tagged/calibration collections at the leaves.

To make a snapshot, we walk the tree depth-first and make a copy of each collection.
- For chained/tagged/calibration collections, just copy the contents of the
 original collection.
- For run collections, create a tagged collection with the contents of the
  original collection.

So the original input collection chain turns into a tree of snapshot collections that looks like:
```
(chain) LSSTCam/calib/2026-07-28
   -> (chain) SNAPSHOT/LSSTCam/calib/16f2aa3b-7f72-4b67-beb4-48a471dc8c3a
       -> (chain) SNAPSHOT/LSSTCam/calib/DM-53546/e7ab8157-222c-4489-98d7-cec4fde1ac29
          -> (calib) SNAPSHOT/LSSTCam/calib/DM-53546/skyFrames_epoch00/sky-u.20260115a/375dc4de-bbfd-4fc0-b9fa-30d45fe9dc32
       -> (tagged) SNAPSHOT/LSSTCam/calib/DM-51669/unbounded/88fe1d3b-17e5-4fd3-9618-9d76ed00a1e0
    ...
```

In the examples above, the generated collection names were like:
**SNAPSHOT**/LSSTCam/calib/**16f2aa3b-7f72-4b67-beb4-48a471dc8c3a**

The three components of the collection name are:
- A prefix `SNAPSHOT/`, which separates these generated collections
 from normal collections named by users.  Many processes built around the Butler involve
 searches for specific collection prefixes, so this prevents the snapshot collections from
 being pulled in by accident.
- The name of the original collection.
- UUIDv5 (deterministic hash) generated from the contents of the collection.  This is the key to several important features.

### The collection hash

The UUID in the collection name is generated by hashing the following data:
- The basic definition fields for the collection (name, type, documentation).
- For run/tag/calib collections, a sorted list of the UUIDs contained in the collection.
- For calibration collections, the timespans associated with each dataset UUID
- For chained collections, the list of child collection names **after conversion to snapshot collections**.

Because of the presence of this hash, two identical snapshot collections will
always have the same name. If anything changes in a collection, it will
generate a different name.

This means that:
- If a child collection of the input collection has not changed since we last
took a snapshot, no new collection is created for that child.  This stops
snapshot collections from proliferating out of control, since in most
collection chains "older" children are static.
- Even if multiple users generate snapshots of collections without any
 coordination, they can still share the same underlying child snapshots.
- You can verify the integrity of a snapshot collection without needing the
JSON file, since everything needed to compute the hash is part of the
definition for the collection stored in the Butler.

### The top level collection

In the examples above, I allowed the user to specify a name for the output top
level collection.  It might be better to generate this collection name with a
hash, to guarantee uniqueness.  However, this might make it harder for users to
remember and understand the collection names they are using. 

If we don't force them to include a hash here, users have to
extra-double-pinky-swear that they won't remake a collection with different
contents after they have transferred it outside its original repository.

The top level collection is created as a single-child chained collection
containing the `SNAPSHOT/` of the input collection.  For the command from the
example:

```
$ butler collection-snapshot create LSSTCam/calib LSSTCam/calib/2026-07-28
```

the structure would be:

```
LSSTCam/calib/2026-07-28   CHAINED
   -> SNAPSHOT/LSSTCam/calib/16f2aa3b-7f72-4b67-beb4-48a471dc8c3a   CHAINED
      -> ... the rest of the snapshot structure based on the input collection
```

This works even if the input collection is not a chained collection.

We can record the expected child collection of the top-level collection in its
its documentation string, to allow the `verify` command to work without needing
the JSON file.  This also lets `load` abort early if there is a mismatch.

### Cleaning up defunct snapshots

Users should never reference any collection starting with `SNAPSHOT/` directly
-- they should always use a top-level chained collection with a
non-`SNAPSHOT/` name.

That means that it is safe to delete any `SNAPSHOT/` collection which is not
ultimately referenced by a non-`SNAPSHOT/` chain.  We could periodically run a
maintenance process to prune these defunct snapshot collections.

### One thing that really will not work well at all

For most of the "manually curated" collections in `LSSTCam/defaults`, this
scheme should work well.  However, there is one collection in that chain which
is a problem: `LSSTCam/raw/all`.

`LSSTCam/raw/all` is a giant RUN collection containing every raw that has ever
been generated by the telescope, and grows by around 200,000 datasets a day.
That means that:
- A new snapshot of it will be created every time a snapshot is taken
- That generated collection will contain an enormous number of rows, bloating
the Butler database
- Any chains involving this collection will create enormous JSON files (which
 eventually won't fit in memory).

These problems could be mitigated with future work to:
- Partition `LSSTCam/raw/all` by `dayobs` instead of having a single
collection.
- Use a JSON file per child collection instead of a single JSON file
  containing the whole tree.
- Add a way to store the snapshot JSON files in the Butler itself, so they can
  be cached instead of needing to recreate them every time.
- More sophisticated support for collection snapshots in the Butler, e.g.
  immutable collections with their hashes stored directly in the Butler database.

That problem does not need to stop us from adopting this workflow for other
collections.  One workaround would be to add another level to the
`LSSTCam/defaults` tree called `LSSTCam/defaults/non_raw` (for lack of a better
name), and take snapshots of that instead of the full `LSSTCam/defaults` with
raws.

## Some implementation details

All of the functionality described here can be implemented with no changes to
the core Butler API.  Many things would work better with additional support in
the Butler data model (e.g. immutable collections), but that is not required
for the minimum viable product.

### File structure

(Note: I have not considered the specifics of this aspect in a lot of detail.
Other JSON layouts or other file formats are possible and probably
better).

Each collection in the snapshot chain is a JSON object encoded on a single
line.  The objects are separated by newlines.  (This is the "JSON Lines"
pseudo-standard.)

Having multiple JSON objects means that we don't have to parse the entire file
in one shot, conserving memory and allowing parallelization.  The collections
should be written depth-first, so that the `load` command can just generate
collections in the order given in the file.

Thus the file (with much of the data omitted) looks something like:

```
{ "name": "SNAPSHOT/LSSTCam/calib/DM-53546/skyFrames_epoch00/sky-u.20260115a/375dc4de-bbfd-4fc0-b9fa-30d45fe9dc32", "type": "CALIBRATION", ...}
{ "name": "SNAPSHOT/LSSTCam/calib/DM-53546/e7ab8157-222c-4489-98d7-cec4fde1ac29", "type": "CHAINED", ... }
{ "name": "SNAPSHOT/LSSTCam/calib/16f2aa3b-7f72-4b67-beb4-48a471dc8c3a", "type": "CHAINED", ... }
{ "name": "LSSTCam/calib/2026-07-28", "type": "TOP LEVEL CHAIN", ...}
```

The specific object structures for the individual collection types could be:

#### Run/tagged

```
{
    "name": "SNAPSHOT/LSSTCam/calib/DM-51669/unbounded/9e3295e9-7b51-4d21-9eb5-c22535649d61",
    "type": "TAGGED",
    "doc": "...",
    "datasets": [ "09968ee8-7197-4c30-90f4-4bc783b2106c", "f02d4944-9da1-4732-9c3a-8b121fcfb50a"]
}
```

#### Calibration

```
{
    "name": "SNAPSHOT/LSSTCam/calib/DM-53546/skyFrames_epoch00/sky-u.20260115a/375dc4de-bbfd-4fc0-b9fa-30d45fe9dc32",
    "type": "CALIBRATION", 
    "doc": "..."
    "datasets": [
        {
            "id": "d043b15f-c4d4-472c-8d7b-fdc16f633970",
            "timespan": [1785259800000000000, 1785346200000000000]
        }
    ]
}
```

The Butler library provides a function for converting the calibration
`Timespan` into a JSON-serializable format.

#### Chained

```
{
    "name": "SNAPSHOT/LSSTCam/calib/16f2aa3b-7f72-4b67-beb4-48a471dc8c3a",
    "type": "CHAINED",
    "doc": "...",
    "children": ["SNAPSHOT/LSSTCam/calib/DM-53546/e7ab8157-222c-4489-98d7-cec4fde1ac29"]
}
```

### Concurrency issues

Provided that the creation and population of each individual collection in the
snapshot chain is wrapped in a Butler database transaction, both the `create`
and `load` commands will tolerate multiple processes working with the same
collections concurrently.  The commands are also idempotent and retryable.

It is not required that the entire chain be created in a single giant
transaction, and it's better if it's not.  In particular, transactions
registering datasets to a calibration collection currently take a global lock
on the table, so these transactions need to be as short as possible.

However, there should be a transaction around each individual collection to
guarantee the atomicity of the operation -- if the collection exists, it must
be populated with the contents described by its hash.

Collections should be created in depth-first order. (The Butler database
constraints will enforce this.) It's OK if the commands fail part-way through
-- any collection that was created is in a valid, complete state because of
the transactions and the fact that all children are guaranteed to be valid
before we attempt to create their parents.

Additionally, `create` and `load` should verify the contents of any collection
they encounter that already exists.  This is just a sanity check since we do
not yet have real support for immutable collections in the Butler, and someone
may have deleted a dataset after it was added to a collection.

### Collection subsetting

With this collection structure, it is possible to take only a subset of the
dataset types in a given collection.  (Or even filter by visit or other data ID
fields.)  This may be useful for workflows like bringing back the outputs from
a DRP run at another site.

With only a subset of the datasets included in the collection hash, you are
guaranteed a collection name distinct from any other subset of the collection.

### Butler APIs useful for implementing this feature

- `Butler.collections.query_info(flatten_chains=True, include_chains=True, include_doc=True, include_summary=True)` will pull up most of the information needed about all of the collections in the chain.  The `include_summary=True` gives
you a list of dataset types that you need to query when looking up the contents
of run/tag/calib collections.
- `Butler.registry.queryDatasetAssociations` can be used to fetch the list of datasets contained in each child collection.  (See the implementation of this function for an example of the Butler "general query" API, which could alternatively be used to fetch the needed data incrementally and faster by omitting the data IDs.)
- `Timespan` is a pydantic model that can be converted to a JSON-serializable format.