# Soft delete by field

Approved by Jason on 2026-11-17, in the project thread "Support soft delete in REST without verbs".

## Problem

`@resource(delete: SOFT)` was one setting that quietly generated different endpoints in each
style, and only Google's AIP-164 defines soft delete. In REST it produced
`POST /books/{book-id}/undelete`, which breaks Zalando rule 141 ("MUST keep URLs verb-free"), so the
starter carried a warning.

## Decision

A nullable field marked `@deleted` is the one switch. `delete: SOFT` is removed; `delete` takes only
`NONE`.

```graphql
type Book @resource {
  id: ID!
  deletedAt: DateTime @deleted # deleted books can be listed and restored
}
```

It means "deleted items can be listed and restored" in every output, each in the way its standard
allows, so no output loses the feature. Without the field, delete is a plain hard delete everywhere.

| Style           | Book with a `@deleted` field                                                                                                                                |
| --------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| AIP HTTP, gRPC  | AIP-164: Delete returns the Book, `:undelete`, `show_deleted` on List                                                                                       |
| REST (Zalando)  | No DELETE. `PATCH /books/{book-id}` with `{"deleted_at": "<time>"}` deletes and `{"deleted_at": null}` restores; `GET /books?show_deleted=true` lists them |
| JSON:API        | No DELETE. The same through its PATCH: `deletedAt` set to a time or `null`; `?showDeleted=true` on the list                                                  |
| GraphQL (Relay) | `deleteBook` returns the Book, `undeleteBook`, `showDeleted` on the list                                                                                    |

REST and JSON:API follow Google Drive (`{"trashed": true | false}` on the item) and Laravel JSON:API
(`deletedAt` set to a date or `null`). With no DELETE, Zalando rule 148 ("after DELETE the resource
must not be accessible on its endpoint") never applies, and there is no verb in a URL. The deletion
time is the one server-set field a REST or JSON:API client may send. A resource with
`update: NONE` and a `@deleted` field still gets that PATCH, taking only the deletion time.
GraphQL mutations are verbs by nature (GitHub's API has `unarchiveRepository` and `reopenIssue`).

## Rules

- A resource has at most one `@deleted` field, of type `DateTime`.
- `@deleted` on a resource with `delete: NONE` is an error: it is never deleted.
- `delete: SOFT` is an error, as any value the argument doesn't allow (`core/directive-uses`).
- Problems removed: `core/timestamps` "A resource with delete: SOFT needs one field marked
  @deleted." and "@deleted needs @resource(delete: SOFT)."; REST's `verb-free-urls` warning for
  undelete.
- Problem added: `core/timestamps` "A resource with delete: NONE is never deleted, so it has no
  @deleted field."

## Delivery

Three pull requests, each reviewed on its own:

1. The notation switch: the field replaces `delete: SOFT`, outputs render as before.
2. REST: PATCH the deletion time, no DELETE, no undelete warning.
3. JSON:API: the same through its PATCH.

Corpus expectations are written by hand from each standard, never copied from engine output.

## Sources

- Zalando RESTful API Guidelines, rules 141 and 148: <https://opensource.zalando.com/restful-api-guidelines/>
- AIP-164 Soft delete: <https://google.aip.dev/164>
- Google Drive, trash and restore: <https://developers.google.com/workspace/drive/api/guides/delete>
- Laravel JSON:API, soft deleting: <https://laraveljsonapi.io/2.x/schemas/soft-deleting.html>
- JSON:API 1.1, deleting resources: <https://jsonapi.org/format/#crud-deleting>
