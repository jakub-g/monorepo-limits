# monorepo-limits

This is just a collection of **hard limits** of various technologies, directly encountered in a day-to-day operations of a large git monorepo.

_Technologies mentioned: git, Node.js, Amazon AWS S3, GitHub.com, GitLab self-hosted_

## Hard limits

### 10 MB: The limit for AWS CloudFront and Google GCP Cloud CDN for on-the-fly compression.

AWS CloudFront [can on-the-fly gzip the files up to 10MB](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/ServingCompressedFiles.html) - which will typically shrink their size to around ~2MB after gzip. Once you exceed 10MB, assets gets sent to customers as-is uncompressed. Which likely means a massive loading time regression.

_No easy workaround:_ just don't allow your assets to exceed 10 MB. 

You can possibly do the compression at build time if you can't make your files smaller, but if done incorrectly, it can backfire.

In case of JS app, you want to split your code to lazy-loadable components dynamically `import()`-ed, and hope your bundle to do the magic.

However, it's extremely easy to make a mistake in your JS code which explodes you bundle size via an innocuous static `import` which brings much more than it looks like.

You should monitor your bundle sizes, on every pull request _and_ in production, and have monitors set up when exceeding thresholds close to 10 MB.

### 20 MB: Total max size of JS/JSON files in a TS project until tsserver stops working

If you have [more than 20 MB of non-TS files (JS, JSON) in a TS project, tsserver will stop working](https://github.com/microsoft/TypeScript/blob/5026c6675cbbfd493011616639595084f899d513/src/server/editorServices.ts#L2758-L2780
).

This is likely a config against misconfigurations like having `dist` visible by TS project.

But it can also happen e.g. when you try to merge a legacy JS codebase into a TS project (merging multiple repos, onboarding an acquisition etc.).

_Workaround_: In `tsconfig.json`, set `compilerOptions.disableSizeLimit: true`.

### 33.8 MB / 45 MB: Github GraphQL payload limit

When making a commit to update some file(s) with GitHub GraphQL API, the limit for payload is 45 MB. However, as the data is transferred base64-encoded, effectively this makes it ~33.8 MB (base64 inflates data size by ~33%).

Note that when updating files with GraphQL API, the entire new version of the modified files is sent to the GitHub backend (not the diff). So when updating a 20MB file (or two 10 MB files, etc.) by editing one line inside, you'll send 20 * 1.33 = 26.6 MB.

_A quick workaround:_ if you update N large files in a single commit, consider doing several smaller commits instead.

### 100 MB: GitHub single file max size

Trying to push a blob bigger than 100 MB will make GitHub reject the push. [source](https://docs.github.com/en/repositories/working-with-files/managing-large-files/about-large-files-on-github)

_No workaround: You need to remove the offending blob from your branch's history, rewriting history with `git rebase`_.

### 512 MB: V8 max string size

You can't allocate a string bigger than 512 MB with V8 (Node.js).

You're most likely to hit this limit when trying to `JSON.stringify()` a large JS object.

_Possible quick fixes:_ 

- writing a non-prettified JSON (no spaces, no newlines) with `JSON.stringify(obj, ..., null)`
- emitting NDJSON (newline-delimited JSON) instead of one large JSON, and updating the consumer code
- trimming the JS object from useless data

### 2 GiB: GitHub packfile max size

[You can't push more than 2 GiB to GitHub at once.](https://docs.github.com/en/get-started/using-git/troubleshooting-the-2-gb-push-limit)

Common scenarios when this happens:

- You have a big repo locally that you want to push to a new remote GitHub repo.
  - _Workaround:_ incrementally push parts of the history, e.g. instead of pushing HEAD, push first the HEAD from N months/years ago, and keep moving forward in history and doing incremental pushes.
- You cloned a large remote repo with `--depth 1`, did some changes locally, and try to push.
  - The problem is that by having shallow clone with `--depth 1`, the HEAD after the clone (the shallow boundary) is not a "real" commit and when trying to push later, git can't properly figure out what it has to push through the standard remote negotiation.
  - _Solution_: to [clone with `--depth 2` to avoid the issue](https://news.ycombinator.com/item?id=43023283), or to unshallow/deepen your clone before pushing.
 
### 5 GB: Amazon AWS single upload limit (inherited GitLab artifacts)

By default, you can't upload more than 5 GB through AWS S3 APIs. The GitLab's `artifacts` inherits this limitation as well, meaning it's not possible to use it to share >5GB of data across CI jobs out of the box.

_Solution_: if you rely on AWS S3 directly, use multipart upload API which allows bigger uploads. 

### ~20 GB: GitHub Actions runner disk size

If you try to write > ~20 GB on disk within GitHub Actions, the workflow will fail due to not enough space. This can happen e.g. when cloning a large repo with `actions/checkout`.

_Workaround 1:_ An obvious first thing to try is to use shallow clone with `depth: 1` or so. But sometimes it's not possible, because you need _some_ recent history in the workflow. But you may not know how many commits exactly you need.

_Workaround 2:_ If fixed `--depth` won't do, you could do shallow clone with `actions/checkout`, and then unshallow partially via a direct git command like `SHALLOW_SINCE=$(date -d "1 month ago" +%Y-%m-%d); git fetch --shallow-since=$SHALLOW_SINCE origin main`. 

_Workaround 3:_ Limit the amount of data transfered by `git fetch` with `--filter=tree:0`, or `--filter=blob:none` etc. (see [this blog](https://github.blog/open-source/git/get-up-to-speed-with-partial-clone-and-shallow-clone/)) or other related options.

## Soft limits

### ~1 MB - 2 MB: a single .ts file size

This is not a set in stone limit, but often TypeScript starts struggling with inferring types in large files, especially if the whole file is just a one large data object (a "JSON" but saved as `.ts`).

_Workaround_: split the large data files; give them explicit types, to avoid TS trying to infer the massive literal type.


