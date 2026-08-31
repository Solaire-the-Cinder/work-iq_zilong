# SharePoint document-library metadata

Use this reference when a user asks about SharePoint library columns or wants
files filtered, counted, grouped, sorted, or compared by metadata.

## Content search and library metadata are different

`ask` and KnowledgeSearch retrieve information from document content and
embedded file properties. SharePoint document-library columns are list-item
fields and must be read through `fetch`.

For example, document text may contain `Document Owner: Sofia Ricci` while the
library's `Owner` column contains `HR Team`. A question about the library column
must use the latter.

Use this route whenever the user mentions:

- metadata, column, field, or property;
- `Owner =`, `Status =`, `City =`, or another named attribute;
- count/group/breakdown by a column;
- earliest/latest by a date column; or
- which documents have a specified column value.

Do not use `ask` as the sole source for those claims. It may summarize file
content only after `fetch` has identified the correct files structurally.

## Canonical resolution recipe

Given:

```text
https://contoso.sharepoint.com/sites/Finance/Shared%20Documents
```

### 1. Resolve the site once

```text
/sites/contoso.sharepoint.com:/sites/Finance
```

Read the returned `id`, which has the composite form:

```text
contoso.sharepoint.com,{siteGuid},{webGuid}
```

Reuse it for the rest of the conversation. Do not construct
`/sites/contoso.sharepoint.com,Finance`; a site name is not a site id.

### 2. Resolve the document library once

```text
/sites/{siteId}/lists?$select=id,name,displayName
```

Choose the list whose `name` or `displayName` matches the target document
library, commonly `Documents` or `Shared Documents`. Retain its `id`.

### 3. Resolve columns before querying items

```text
/sites/{siteId}/lists/{listId}/columns?$select=name,displayName,indexed,hidden
```

Build a mapping from the user-facing `displayName` to the internal `name`.

| User asks for | `/columns` may return | Use in item fields |
|---|---|---|
| Owner | `displayName: Owner`, `name: Owner` | `fields.Owner` |
| Review Date | `displayName: Review Date`, `name: Review_x0020_Date` | `fields.Review_x0020_Date` |
| Document Type | `displayName: Document Type`, `name: DocumentType` | `fields.DocumentType` |

Match names case-insensitively and ignore spaces/underscores when comparing.
Use the internal `name` exactly as `/columns` returns it — spaces and special
characters are encoded (e.g. `Review_x0020_Date`). Never assume the internal
name from the display label; read it from `/columns`, and disclose a surprising
mapping such as: “The library's Review Date column is stored internally as
`Review_x0020_Date`.”

If no column matches, stop and say that the library has no requested column.
List the relevant available display names from `/columns`; do not guess,
substitute, or relabel another field.

### 4. Read item fields

Use a page size accepted by the current tool. `$top=100` is an example request,
not proof that the response is complete.

Full fields:
```text
/sites/{siteId}/lists/{listId}/items?$expand=fields&$top=100
```

Selected fields:

```text
/sites/{siteId}/lists/{listId}/items?$expand=fields($select=FileLeafRef,Owner,Status)&$top=100
```

Columns on a `listItem` live below `fields`. A bare list-item
`$select=Owner,Status` is invalid.

## Grounding contract

`/columns` is authoritative for what the library carries. GET it before
answering about any property; if the property is not in that set, no further
retrieval will produce it.

Before reporting each metadata value:

1. Confirm that `/columns` contains the requested column.
2. Confirm that this item's tool result literally contains the internal field
   name and value.
3. Keep absent, empty, and unreadable distinct:
   - absent column: “This library has no `<X>` column”;
   - present field with an empty value: report it as empty;
   - failed call: “Could not read this value because the call failed.”
4. Never derive a column value from a filename, folder path, URL segment,
   document text, or a related field.
5. Never relabel another field as the requested field.

### Absent-field protocol

Treat schema absence and empty item values as different findings:

- If `/columns` does not contain the requested property, state in the first
  sentence that the library has no such column. Do not show a per-file table for
  that property. You may name nearby columns only when clearly labelled as
  different data and ask whether the user wants one of them instead.
- If the column exists, report an item as empty only when that item's returned
  `fields` contains no value for the resolved internal name.
- Say the column is empty for every item only after retrieving a COMPLETE
  candidate set. For a partial set, say only that the retrieved items were empty
  and disclose the coverage limitation.

Do not continue searching for a column that `/columns` proves absent. Do not
substitute another field or invent example values.

### Never substitute one field for another

- `Modified` / `Modified By` is not checkout state, and not review activity.
- `publication.level` is not a sensitivity label, and not checkout status.
- `Created` / `Created By` is not an approval record.
- any date column is not a view or access count.

Examples of invalid grounding:

- `Seattle.docx` does not prove `City = Seattle`.
- `/Projects/Project_Beta/file.pdf` does not prove `Owner = Project Team B`.
- `lastModifiedDateTime` is not `Review Date`.

If you must offer an inference, label it explicitly and keep it separate from
the values read from SharePoint.

## Scope and denominator

Use the population the user requested. By default, that is the target document
library or folder, not an inferred subset such as items with `Owner` populated.
If the user explicitly asks for a subset, resolve that subset's column through
`/columns` and apply it without substituting another field.

Name the denominator whenever you report a count or percentage. State a total
or percentage only after retrieving a COMPLETE candidate set. If retrieval is
partial, report only the number of items retrieved or matched as a lower bound
and describe what remains unread; never use an unrelated column to manufacture
a denominator.

## Filtering and sorting

Use the confirmed internal column name in server-side queries. Escape every
user-controlled OData string literal before URL encoding it: double each
apostrophe (`O'Brien` becomes `O''Brien`), then URL-encode the complete query
value.

```text
/sites/{siteId}/lists/{listId}/items?$expand=fields&$filter=fields/Status%20eq%20%27In%20Progress%27&$top=100
```

If the tool reports that the column cannot be filtered or ordered because it is
not indexed, do not retry cosmetic variants. Remove the rejected operation,
retrieve list items and their `fields` through supported paging, then process
the complete set client-side. If a complete set cannot be retrieved, provide a
qualified partial result instead of a total, percentage, or extrema claim. Do
not switch to `ask` or KnowledgeSearch for library-column values.

## Enumerating a library

### 1. Enumerate list items directly

```text
/sites/{siteId}/lists/{listId}/items?$expand=fields&$top=100
```

When a response includes `@odata.nextLink`, it is incomplete. Follow the
returned continuation in the form accepted by the current `fetch` tool; treat
it as opaque and do not construct a `$skiptoken` yourself. Continue until no
next link remains. If the tool rejects the returned continuation, stop paging
this way and continue with drive traversal below. If traversal is also
unavailable, report partial coverage rather than trying unsupported pagination
variants.

### 2. Use drive traversal only to discover candidates

If list-item paging is unavailable but the host permits drive traversal,
resolve the library's drive and root item first:

```text
/sites/{siteId}/lists/{listId}/drive?$expand=root
```

Use the returned drive `id` and `root.id` as `driveId` and `folderItemId`.
If the drive response succeeds but does not expand `root`, read
`/drives/{driveId}/root?$select=id` once to resolve the root item. If that call
is explicitly policy-denied, stop and report the denial.
For a group-backed site that was resolved through its group, use the equivalent
`/groups/{groupId}/drive?$expand=root` route described in
[SharePoint and OneDrive](sharepoint-work-iq.md).

List folder children with their associated list-item fields when the host
accepts the nested expansion, and follow every returned continuation:

```text
/drives/{driveId}/items/{folderItemId}/children?$expand=listItem($expand=fields)
```

A child is a `driveItem`; it does not by itself provide the document-library
column values. If the collection rejects the nested expansion, list the children
without it and request each file candidate's associated `listItem` and fields:

```text
/drives/{driveId}/items/{driveItemId}/listItem?$expand=fields
```

If that response supplies a list-item id but not its fields, read the list item
through the resolved site and list:

```text
/sites/{siteId}/lists/{listId}/items/{listItemId}?$expand=fields
```

Use only shapes accepted by the current WorkIQ tool. If the relationship or
fields cannot be read, disclose that gap; do not filter or aggregate metadata
from drive-item names, paths, or other facets.

Every child with a `folder` facet is a container. Repeat the children request
for that folder's item id, exhaust its continuation pages, and continue until
every discovered subfolder has been traversed. Folder traversal is incomplete
if any discovered folder or page remains unread.

### 3. Target a single known list item

```text
/sites/{siteId}/lists/{listId}/items/{id}?$expand=fields
```

Use this only for a known item, not to probe an id range.

## Completeness rules

A candidate set is COMPLETE only after every page has been retrieved, every
discovered subfolder has been traversed, and every candidate needed for the
answer has a successful list-item `fields` response. An `@odata.nextLink`
always means more pages remain. A page whose row count equals the requested
`$top` is not, by itself, proof of either completeness or truncation; rely on
continuation metadata and, when available, a tool-returned count known to cover
the same scope. For example, a folder's `childCount` applies only when the
requested scope is exactly that folder's immediate children.

From an incomplete set, do not state a whole-library total, percentage, most,
least, earliest, latest, or all-empty conclusion. Report the retrieved coverage
and give matching counts only as lower bounds. Never present a partial set as
complete. A request for a count, percentage, extrema, or all-empty conclusion
requires a complete set and therefore overrides the generic 2-3-page cap in
[Fetch](fetch-work-iq.md); if exhausting the pages is impractical or blocked,
return a qualified partial result instead.

### Answer delivery and arithmetic

Put the requested answer and its coverage qualification in the response. If the
host supports file artifacts, they may supplement a large result but must not
replace the answer; do not assume a filesystem path or attachment capability.

Before reporting a breakdown and total, recompute each group from the final
retrieved set, verify that the groups sum to the stated total, and verify that
the total matches the complete candidate set. Use a computation tool only when
the current host exposes one; otherwise check the arithmetic directly.

## Per-result status codes

Do not rely on an outer success flag when the response also contains per-request
results. Inspect each returned status and error message.

For a batched `fetch` containing N explicit `entityUrls`, reconcile N requested
URLs with N per-URL results. Count successful URLs, recover or disclose failed
URLs, and never treat a failed URL as an empty entity.

For one collection URL, first verify that collection request succeeded. Then
reason about its returned rows and `@odata.nextLink` separately. Collection rows
are not requested URLs, so do not compare row count with the number of URLs in
the batch.

## SharePoint error handling

Specific error evidence takes precedence over a generic HTTP status rule:

- `Access denied for path: <X>` and `Access denied for GET path: <X>` are
  explicit policy denials. Stop and report the returned denial; do not
  re-address the same target or try alternate paths to bypass the policy.
- If the response identifies a query shape or parameter as unsupported, stop
  using that shape even when the status is 500. Do not retry different quoting,
  casing, or ordering unless the error identifies the request formatting as the
  problem.
- If a 500 response has no specific unsupported-shape explanation, retry that
  same target once in isolation. If it still fails, report it as unreadable.
- If a call fails with no diagnostic detail, check the request format and known
  identifiers, correct a demonstrated problem if present, and retry once. Do
  not claim a status code or cause that the tool did not return.
- A failed call is not an empty folder, absent column, or empty field.

## Batching

Batch related, known-good reads when the current tool schema permits it. Stay
within any limit reported by the tool or host. If a batch is rejected for size,
split it into smaller batches; if one target causes a batch failure, isolate
that target. Apply the per-URL reconciliation rules above to every explicit
`entityUrls` batch.