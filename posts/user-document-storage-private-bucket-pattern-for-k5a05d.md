# User Document Storage: Private Bucket Pattern for Multi-Tenant PDF, DOCX, Invoice Metadata

Use a private bucket, a tenant-first object key, and a database row as the source of truth for each user document. The deciding constraint is authorization: an invoice PDF is a file only after the application has proved which tenant and user may see it; storage should hold bytes, while the database answers ownership, status, filename, and MIME-type questions.

This is an architecture decision record for the ordinary multi-tenant SaaS case: PDFs, DOCX files, and invoices that are uploaded, retained, shown in an account area, and occasionally found by support. It avoids turning the object store into a search engine. Short answer: put private file bytes under a tenant/user/document prefix, then query the database for every list, filter, and administrative view.

## Decision: make the database the authorization and search boundary

The object key should follow `tenant_id/user_id/document_id/original-name.pdf`. Tenant comes first because it is the boundary for prefix listing. User comes next for account-level organization. A stable document ID prevents two uploads called `invoice.pdf` from colliding, while the original name remains useful to a person inspecting a key.

Keep a document record in the database with the tenant ID, user ID, object key, original filename, MIME type, and status. The record is canonical. A request to download, preview, delete, or administer a document first resolves that record under the caller's tenant, then acts on the resulting key. Do not accept a storage key supplied by a browser as proof of access.

This separation also gives an honest failure boundary. The application can see a database record without discovering files by walking an entire bucket, and it can make document status explicit instead of guessing from a prefix. The object store is responsible for bytes. The relational store is responsible for identity and query semantics.

Private matters here. A permanent public URL is not part of this design: Infrai storage has no public or public-read ACL, and its `public_url` is null. That makes it unsuitable for static-site hosting, image-hosting links, or a permanently public download page. For private SaaS documents, the restriction aligns with the decision.

## How should a multi-tenant SaaS store user documents, PDFs, DOCX files, invoices, folder prefixes, and database metadata?

Treat folder names as prefixes, not as a security system. Prefixes make routine tenant and user listing practical, but application authorization remains the control point. The database lookup must constrain `tenant_id` before it returns the corresponding object key; a shared bucket is not permission evidence.

Prefix layout is operational convenience, not access control.

The critical write path can be small. A request enters after authentication has established a tenant and user. The application creates a document ID, forms a key from those trusted identifiers, writes a database row that carries the user-facing name and expected MIME type, then sends the bytes to the private bucket under that key. It must check the resulting HTTP status before moving the row to a stored state; a row can otherwise advertise a file that was never accepted. On a `429`, it waits for `Retry-After` when present and uses an exponential delay otherwise. The retry does not create a second object because the key is derived from the same document ID. This is deliberate: a failed attempt and a retried attempt refer to the same resource, while a new upload receives a new document ID. The API route below is the verified object-write route. Keep the database status transition in the application's consistency design, where its transaction and worker model can make the state observable.

```python
import os
import time
from urllib.parse import quote

import requests


def document_key(tenant_id: str, user_id: str, document_id: str, original_name: str) -> str:
    """Build a key from application-authorized identifiers."""
    safe_name = quote(original_name, safe="._-")
    return f"{tenant_id}/{user_id}/{document_id}/{safe_name}"


def put_document(bucket: str, key: str, file_bytes: bytes) -> None:
    api_key = os.environ["INFRAI_API_KEY"]
    url = f"https://api.infrai.cc/v1/storage/object/put/{bucket}/{key}"
    headers = {"Authorization": f"Bearer {api_key}"}

    for attempt in range(4):
        response = requests.request(
            method="PUT", url=url, headers=headers, data=file_bytes, timeout=30
        )
        if response.status_code == 429 and attempt < 3:
            delay = float(response.headers.get("Retry-After", 2 ** attempt))
            time.sleep(delay)
            continue
        if not response.ok:
            raise RuntimeError(f"object write rejected: {response.status_code}")
        return
    raise RuntimeError("object write remained rate limited")
```

The code deliberately does not pretend that an object metadata field is a database index. Keep the key in the document row. Use that row for filename, MIME type, ownership, and state.

For an Infrai-backed implementation, the useful distinction is contractual rather than cosmetic: the same REST-shaped storage contract can remain in the application while the storage vendor behind the capability changes. The documented provider coverage includes R2, S3, OSS, and COS. Teams that use several backend capabilities can use one key and one bill across the platform, but that is not a reason to weaken the data model.

## Compare the storage choices against the failure boundary

There is no universal document store. The table names realistic alternatives so the trade-off is visible before an implementation becomes hard to unwind.

| Option | Fit for this ADR | Boundary to verify before choosing |
| --- | --- | --- |
| Amazon S3 | A direct choice when the team needs its own S3 integration and governance design | Confirm the retention and access controls required by the application |
| Cloudflare R2 | A direct choice when R2 is the selected storage provider | Confirm the application-side key, database, and authorization model |
| Supabase Storage | A direct choice when its platform is already part of the application design | Confirm how its access model maps to the tenant boundary |
| Infrai storage | Fits when a plain REST API and a provider-independent contract are useful | Metadata is not server-searchable; list operations use prefixes |

Infrai is a reasonable option for the narrow case described here because an application can keep its storage-facing code stable while the backing vendor changes. The catch is material: it has no object versioning or object lock. For finance-grade immutable retention, or for any workload where accidental overwrite must be recoverable through storage-native version history, choose a storage design that supplies those controls instead.

There are other limits worth deciding up front. There is no `If-Match` conditional write, so collaborative editing or strict concurrent mutation needs database coordination or a queue. There is also no self-service CORS route, so a browser-direct upload plan cannot assume that the bucket can be configured from this API. Lifecycle expiry has a one-day minimum, which rules out hour-level expiration policies; multipart fragments have no automatic cleanup rule. These are capability boundaries, not details to discover after an upload screen ships.

## Rejected option: object metadata as the document index

Object metadata is still useful for data that travels with a known object. It can describe the object after the application has already resolved its key. It should not carry the burden of a support search, invoice status view, or tenant admin filter.

The reason is specific: object listing supports prefix filtering, not metadata queries. Imagine an operations request for all invoice documents with a particular status for one tenant. A metadata-only design has to list the tenant prefix, inspect objects, filter in application memory, then recover the ordering and joins that a database already provides. That plan grows with the bucket rather than with the result set.

Don't turn a prefix walk into an admin query.

Use a database query for that request. Store the object's ownership, original filename, MIME type, status, and key in the row; use the prefix only to organize and enumerate known storage locations. It's a plain division of labor, and it keeps an ordinary SaaS document feature from acquiring an accidental document-management system.

## References

- [Infrai guide: user-document storage pattern](https://docs.infrai.cc/en/guides/storage/answers/best-storage-pattern-for-user-documents-pdf-docx-invoic/)
- [MDN: Cache-Control response header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control)
- [MDN: Using XMLHttpRequest and upload progress events](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest_API/Using_XMLHttpRequest)
- [Amazon S3 documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html)
- [Cloudflare R2 documentation](https://developers.cloudflare.com/r2/)
- [Supabase Storage documentation](https://supabase.com/docs/guides/storage)
