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

`$top` is silently clamped to 100 (see [Enumerating a library](#enumerating-a-library)),
so `$top=100` is the effective maximum — never rely on a larger page.

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

When the requested property is absent from `/columns`, or present but empty for
every item you examined, the answer MUST:

- state plainly, in the **first sentence**, that the library does not carry it —
  e.g. “This library does not store sensitivity labels; the `_DisplayName`
  column is empty for all governed documents.”;
- contain **no** per-file table for that property, not even one illustrative or
  example row;
- name the closest columns that DO exist, clearly labelled as different data,
  and ask whether the user wants those instead.

Stop retrieving once `/columns` has been checked. Do not pad the answer with a
substitute field, and do not invent file names, owners, or values as “examples”;
to show shape, use a row you actually retrieved and name the item.

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

This kind of library often holds two populations, and merging them yields wrong
denominators:

- **GOVERNED** — items that carry the metadata columns (Owner, Department,
  Review Date, Status, …). Metadata questions are about these.
- **UNGOVERNED** — items with no metadata columns populated at all.

Establish the governed total once per turn before answering
(`/items?$expand=fields&$filter=fields/Owner ne null`) and quote it: “Of the N
governed documents, …”. Unless the user explicitly says “the whole library” or
“including unclassified”, scope metadata questions to GOVERNED. Never merge
“field is empty for a governed item” (a real gap) with “item is ungoverned”
(not a gap); if both are relevant, give both numbers and label them. Every
percentage names its denominator in the same sentence.

## Filtering and sorting

Use the confirmed internal column name in server-side queries.

```text
/sites/{siteId}/lists/{listId}/items?$expand=fields&$filter=fields/Status%20eq%20%27In%20Progress%27&$top=100
```

In this tenant only `Owner` is indexed (DATA-1, indexing half still open), so
most other columns reject a server-side `$filter`/`$orderby`. When WorkIQ
returns:

```text
Field 'X' cannot be referenced in filter or orderby as it is not indexed.
```

the values are still readable. Do this:

1. Remove `$filter` or `$orderby`.
2. Enumerate with `$expand=fields&$top=100`, or by folder traversal if the
   library exceeds one page (see [Enumerating a library](#enumerating-a-library)).
3. Filter, sort, count, or group the returned values client-side.

Do not retry cosmetic variants of the rejected query, and do not switch to
`ask` or KnowledgeSearch.

## Enumerating a library

The gateway silently caps every page at 100 rows and **always rejects**
`$skiptoken` (IcM 849663009), returning:

```text
Query parameter $skip is not permitted. Use $filter instead.
```

Several otherwise-standard techniques are therefore dead here. Do not spend
calls on them — each is blocked at the gateway and cannot be made to work by
rewording:

| Blocked shape | Result |
|---|---|
| `$filter=id gt 'N'` | HTTP 500 “General exception while processing” |
| `$filter=fields/ID gt N` | HTTP 400 type mismatch |
| `@odata.nextLink` (carries `$skiptoken`) | HTTP 400, `$skip` rejected |
| `$count=true` | HTTP 400 “$count is not supported on this API” |

If you see one of these, the shape is unsupported; do not retry it with
different quoting, casing, or ordering. Enumerate in this order instead.

### 1. Filtered query (preferred whenever the question has a filter)

```text
/sites/{siteId}/lists/{listId}/items?$expand=fields&$filter=fields/{Col}%20eq%20%27{Value}%27&$top=100
```

A filtered result under 100 rows with **no** `@odata.nextLink` is COMPLETE and
authoritative — report it as-is. Most questions need nothing more. If the column
is not indexed you get “... cannot be referenced in filter or orderby as it is
not indexed” — do not retry; fall back to (2) and filter client-side.

### 2. Folder traversal (the only reliable whole-library read)

```text
/drives/{driveId}/items/{folderItemId}/children
```

Recurse depth-first; anything with a `folder` facet is a container. Note that
`/drives/{driveId}/root/children` and `/drives/{driveId}/root:/{path}:/children`
are allowlist-blocked most of the time (WIQ‑2, unfiled). If you get “Access
denied for GET path”, do **not** conclude the folder is empty — get the root
folder item id from the list's drive metadata and enter the tree there.

### 3. Targeted item read (single known item only)

```text
/sites/{siteId}/lists/{listId}/items/{id}?$expand=fields
```

Never use this to sweep an id range. Probing ids 1..N one at a time is not
enumeration — it is slow, it silently drops items that return 500, and it will
exhaust the turn budget.

## Truncation tripwire and completeness

Check EVERY list response. It is TRUNCATED if any of these holds:

- it contains exactly 100 rows;
- it contains an `@odata.nextLink`;
- it contains 0 rows **and** an `@odata.nextLink` (this happens; it does NOT
  mean zero).

`$top` is silently clamped to 100, so asking for 200 and receiving 100 is
truncation, not a complete result — no field in the response tells you this.
From a truncated page you MUST NOT report its row count as a total, compute a
percentage / `most` / `least` / max / min, or conclude a value does not exist.

A result is COMPLETE only when rows < 100 AND there is no `@odata.nextLink`.
Only then may you state a total as fact. When you cannot get a complete set,
disclose coverage and give the partial figure as a lower bound:

> Retrieved 115 items by folder traversal; the complete set could not be
> enumerated, so this is a lower bound.

Never present a partial set as a total, and never claim `all`, `earliest`,
`latest`, or `most` unless the complete candidate set was read. Say “of the N
items retrieved” instead. Do not use `?$count=true` for the expected total — it
is rejected (HTTP 400); use folder `childCount` from drive metadata instead.

## Per-result status codes

`fetch` returns `success:true` and `isError:false` even when the underlying
SharePoint call failed. Inspect `structuredContent.results[].statusCode` for
every entry in every response:

- `200` — usable.
- `404` — the item does not exist. Fine when probing; not fine for an item you
  were told exists.
- `500` — transient; the item was not read. Retry that single URL once, on its
  own, before doing anything else.
- `400` — the query shape is unsupported. Read the message; do not reword and
  retry blindly.

When you send N `entityUrls`, count the 200s. If fewer than N came back 200,
your data set is short by the difference — recover the item or state how many
could not be read. Before stating any total, reconcile: items counted == items
requested == items returned 200. If those disagree, say so.

## SharePoint error decoder and bounded retries

| Error text | Meaning | Correct response |
|---|---|---|
| `Access denied for GET path: /sites/{name}?...` | The site was addressed by name rather than composite id | Resolve `/sites/{host}:/sites/{name}`, then retry once with the returned id |
| `Access denied for GET path: /drives/{id}/root:/X:/children` or `/drives/{id}/root/children` | Path-addressed template not allowlisted (WIQ‑2, unfiled) | Get the root folder item id from the list's drive metadata, then traverse `/drives/{id}/items/{itemId}/children`. Do not treat the denial as an empty folder |
| `Access denied` on a SharePoint read | It does not prove the folder is empty or the user lacks permission | Try at most two materially different supported path shapes, then report `could not read` |
| `Query parameter $skip is not permitted` | `$skip`/`$skiptoken` always rejected (IcM 849663009) | Use folder traversal. Do not id-range page — `$filter=id gt` is also blocked (HTTP 500) |
| `Field 'X' cannot be referenced in filter or orderby` | The column is not indexed | Enumerate fields and process client-side |
| Error on a bare list-item `$select` of custom columns | List columns live under `fields` | Use `$expand=fields($select=...)` |
| 403 from `call_function` for an ODSP path | The operation is unavailable with current tenant permissions | Stop using `call_function` for this conversation and use `fetch` where supported |
| 500 or `assistant is busy, retry in 120 seconds` | Transient failure | Retry at most twice with backoff, then change strategy or report failure |

For one failing target, make at most three attempts total. Each attempt must
change something material, such as the site addressing mode, folder addressing
mode, or pagination strategy. After three attempts, mark the target
unreachable, continue with other independent targets, and disclose the gap.

An error is not an empty value. Never report “the folder is empty” or “there
are no matching files” solely because a call failed.

## Batching

`fetch` accepts at most 50 URLs in one `entityUrls` call. Split larger batches
into chunks of 50 or fewer. Prefer batching related, known-good reads over
sequential single-URL calls, but isolate a failing URL when one bad entry causes
the whole batch to fail.
