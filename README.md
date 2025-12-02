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

### 2 GiB: GitHub packfile max size

[You can't push more than 2 GiB to GitHub at once.](https://docs.github.com/en/get-started/using-git/troubleshooting-the-2-gb-push-limit)

Common scenarios when this happens:

- You have a big repo locally that you want to push to a new remote GitHub repo.
  - In that case you can try to incrementally push parts of the history, e.g. instead of pushing HEAD, push first the HEAD from N months/years ago, and keep moving forward in history and doing incremental pushes.
- You cloned a large remote repo with `--depth 1`, did some changes locally, and try to push.
  - The problem is that by having shallow clone with `--depth 1`, the HEAD after the clone (the shallow boundary) is not a "real" commit and when trying to push later, git can't properly figure out what it has to push through the standard remote negotiation.
  - The solution is to [clone with `--depth 2` to avoid the issue](https://news.ycombinator.com/item?id=43023283), or to unshallow/deepen your clone before pushing.
