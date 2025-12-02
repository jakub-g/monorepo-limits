# monorepo-limits

_Technologies mentioned: git, Node.js, Amazon AWS S3, GitHub.com, GitLab self-hosted_

Hard limits.

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
 
### 5 GB: Amazon AWS single upload limit (inherited GitLab artifacts)

By default, you can't upload more than 5 GB through AWS S3 APIs. The GitLab's `artifacts` inherits this limitation as well, meaning it's not possible to use it to share >5GB of data across CI jobs out of the box.

The solution if you rely on AWS S3 directly is to use multipart upload API.

### ~20 GB: GitHub Actions runner disk size

If you try to write > ~20 GB on disk within GitHub Actions, the workflow will fail due to not enough space. This can happen e.g. when cloning a large repo with `actions/checkout`.

An obvious fix is to use shallow clone with `depth: 1` or so. But sometimes it's not possible, because you need _some_ recent history in the workflow. But you may not know how many commits exactly you need.

In that case you could do shallow clone with `actions/checkout`, and then unshallow partially via a direct git command like `SHALLOW_SINCE=$(date -d "1 month ago" +%Y-%m-%d); git fetch --shallow-since=$SHALLOW_SINCE origin main`. 

Another option: `git fetch` with `--filter=tree:0`, or `--filter=blob:none` (see [this blog](https://github.blog/open-source/git/get-up-to-speed-with-partial-clone-and-shallow-clone/)) or other related options.
