# monorepo-limits

### 33.8 MB / 45 MB: Github GraphQL payload limit

When making a commit to update some file(s) with GitHub GraphQL API, the limit for payload is 45 MB. However, as the data is transferred base64-encoded, effectively this makes it ~33.8 MB (base64 inflates data size by ~33%).

Note that when updating files with GraphQL API, the entire new version of the modified files is sent to the GitHub backend (not the diff). So when updating a 20MB file by editing one line inside, you'll send 20 * 1.33 = 26.6 MB.

### 100 MB: GitHub single file max size

Trying to push a blob bigger than 100 MB will make GitHub reject the push. [source](https://docs.github.com/en/repositories/working-with-files/managing-large-files/about-large-files-on-github)

### 512 MB: V8 max string size

You can't allocate a string bigger than 512 MB with V8 (Node.js).

You're most likely to hit this limit when trying to `JSON.stringify()` a large JS object.

Possible quick fixes include: writing a non-prettified JSON (no spaces, no newlines) with `JSON.stringify(obj, ..., null)`; emitting NDJSON (newline-delimited JSON) instead of one large JSON, and updating the consumer code; trimming the JS object from useless data etc.
