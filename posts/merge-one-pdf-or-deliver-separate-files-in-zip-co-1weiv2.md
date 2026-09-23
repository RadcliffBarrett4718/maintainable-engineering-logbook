# Merge One PDF or Deliver Separate Files in ZIP: Contract Audit Retention Costs

For a media contract packet, deciding whether to merge into one PDF or deliver separate files in a ZIP changes the bill: retained bytes multiplied by retention time matter more than the moment of download. Suppose a hypothetical packet contains four 2 MB source PDFs and a 9 MB merged rendering. Keeping both representations for 1,000 packets consumes 17 GB before replicas and versions; keeping only the four originals consumes 8 GB. Those are arithmetic examples, not measured compression ratios or prices. The expensive decision is which representation remains authoritative after signatures and audit records exist.

TL;DR: Deliver one PDF when the signer must review a single, fixed sequence and the signed artifact is that exact rendering. Deliver separate files in a ZIP when independent originals must retain their own identities or different recipients need different subsets. Treat the ZIP as transport, not as the signed artifact; record hashes and signing events against the actual files. Retain a merged copy only when its exact rendered bytes have evidentiary or operational value.

## Should you merge one PDF or deliver separate files in a ZIP?

A packet may contain a talent agreement, a release, a rate sheet, and an exhibit. Rendering them into one PDF can require a second full-size object, plus storage for intermediate output if the rendering pipeline persists it. The equation to model is retained source bytes + retained merged bytes + retained versions + audit records, multiplied by the number of packets and the retention period. A ZIP often compresses already compressed PDF content poorly; its meaningful savings here come from avoiding a second retained rendering, not from assuming a large compression ratio. Test with your own inputs.

There is also compute: merge time, retries, and the cost of generating a faithful combined view each time a reviewer asks for it. Measure median and tail render time on packets with long exhibits, embedded fonts, and mixed page sizes. A storage saving can turn into repeated render work if every audit request rebuilds the same packet.

That changes the decision. Render once or repeatedly? Count both paths.

For signed media contracts, render fidelity has a stricter meaning than a preview looking similar. Page order, page boundaries, signature placement, and the bytes associated with the signed record need a stable interpretation. ISO 32000-2 defines the PDF format; it does not make an arbitrary newly merged PDF identical to the inputs for audit purposes. Record which exact bytes were presented and signed, and preserve the signing evidence under the applicable retention policy.

## When does merging change the evidence?

Decide what the signature covers before choosing a download container. If a signer consents to a combined 40-page packet, retain that exact signed PDF and its digest. Don't regenerate it from four source documents later and assume the result is interchangeable: even a visually equivalent rendering may have different bytes, metadata, or page structure. If each contract is signed separately, preserve each signed file and its digest; a merged convenience copy is a derivative. Label it as such in the manifest.

A manifest should identify the packet ID, ordered document IDs, each file's media type and cryptographic digest, the rendering revision, and the relationship between source and derived files. Keep audit events such as presentation, signature, and download associated with the relevant artifact ID and timestamp. This matters when a release is corrected after the rate sheet has already been signed. Do not quietly replace the old release inside an existing packet; create a new revision and keep the earlier evidence according to policy.

ZIP preserves file boundaries for handoff, but it does not supply a viewing order or prove that every extracted file was reviewed. A client can sort names differently, and a recipient may open only one item. If review order is a requirement, enforce it in the signing flow and record what was presented.

The archive is delivery packaging. Nothing more.

## Where should the render boundary sit?

Use a single PDF for the signer-facing workflow when one ordered artifact is the unit of approval. Produce it before the signing event, validate page count and expected signature locations, and bind the audit record to its digest. Retain the inputs too only when a real business or compliance obligation requires them. Otherwise, define a documented retention rule for derived and source artifacts; don't let an unspecified cache become permanent evidence storage.

Choose separate signed PDFs when approvals can occur independently, or when a broadcaster receives the release but accounting receives the rate sheet. Build the ZIP at delivery time from immutable signed files and a manifest. The archive can be regenerated; the signed files and their audit records cannot be casually replaced. Use access controls per recipient so an archive assembled for one party does not leak a document intended for another.

This is a trade-off, not a universal format rule. Storing the merged signed artifact increases retained bytes but avoids rebuilding the exact thing that was approved. Keeping only separately signed originals lowers duplication but requires a reviewer to reconstruct the packet context from the manifest and event log. In either model, make deletion schedules explicit and check them against contractual and jurisdiction-specific retention requirements with counsel. A policy that retains originals for a shorter period than signed derivatives needs to say what an investigator can still verify after the originals expire. A policy that keeps every preview forever needs to justify why the extra renderings are evidence rather than disposable copies.

## How do you test and operate the choice?

Exercise packets with rotated pages, missing fonts, duplicate filenames, a revised exhibit, and a signing attempt interrupted between upload and audit-record persistence. Assert that a retry cannot create two authoritative revisions for one signing action. Compare digests at ingestion and before delivery; log packet ID, artifact ID, render revision, outcome, and elapsed render time without logging document contents. Alert on missing audit events and failed deliveries rather than treating a successful archive creation as proof of receipt.

Start with a representative corpus and measure retained bytes per packet, render latency at the tail, failed-render rate, and the time needed to answer an audit request. Include version and replica overhead in the estimate. A small reduction in object count may be irrelevant if a long exhibit dominates bytes; a rare rendering mismatch can matter far more than the storage difference.

Measure the exception cases. They matter most.

Finally, decide what you deliberately stop keeping. If an unsigned merged preview is reproducible and has no independent evidentiary role, expire it after the signing workflow. The cost is that a later investigation must reconstruct the preview from retained inputs and a recorded renderer revision, and the reconstruction may not be byte-identical. If that loss is unacceptable, retain the exact preview instead and account for its bytes.

## References

- ISO 32000-2, Portable Document Format: https://www.iso.org/standard/75839.html

## Further reading

- https://www.iso.org/standard/75839.html
