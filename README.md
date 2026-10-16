# Dockerignore vs Gitignore Pattern Tester

Paste one ignore file and a list of paths and see, path by path, whether git's .gitignore rules and Docker's .dockerignore rules each exclude it and which line decided, with the anchoring, trailing slash and re-include differences named.

**Live demo:** https://0xelitesystem.github.io/dockerignore-vs-gitignore-pattern-tester/

## Use

1. Paste the ignore file into the left box. The git column reads it as a single `.gitignore` at the repository root; the Docker column reads it as the `.dockerignore` of a local build context.
2. Paste the paths you care about into the right box, one per line, relative to the root. End a directory with `/`, because a git pattern with a trailing slash only matches directories. Backslashes are read as `/`.
3. Optionally type the Dockerfile path. The page then says which ignore file Docker reads for that Dockerfile.
4. Read the table: for each path, git's verdict and deciding line, Docker's verdict and deciding line, and whether they differ. Under every differing row is one sentence naming the rule pair behind it (G1 to G7 for git, D1 to D8 for Docker, all listed on the page). Tick "Show differences only" to hide the rows that agree.

The page opens on the worked example:

```
target
node_modules
!node_modules/keep.txt
*.md
!README.md
foo/
*.log
```

Five of its eleven paths differ: `src/target/b.txt`, `docs/x.md` and `logs/build.log` (git matches an unanchored pattern at any depth, Docker anchors every pattern at the context root), `node_modules/keep.txt` (Docker re-includes it through the `!` line, git cannot because the parent directory is excluded) and a regular file `foo` (Docker drops the trailing slash of `foo/`, git keeps it as directories only).

For the git side alone, `git check-ignore -v <path>` gives the same answer as the git column and is the command to use. What it cannot show is the Docker reading of the same file next to it.

### When this is useful

- You copied `.gitignore` to `.dockerignore` and the image is bigger than expected, or a build step cannot find a file it needs.
- A `!` exception works in Docker but not in git, or the other way round, and you need to show a teammate which line decides.
- You are reviewing a pull request that edits either file and want to see which paths change verdict.
- You maintain several Dockerfiles and want to confirm which ignore file applies to each one.

## What it models

**git**, implemented from the PATTERN FORMAT section of [gitignore(5)](https://git-scm.com/docs/gitignore) (the page showed "gitignore last updated in 2.55.0" when read for this build):

- Blank lines and `#` comments match nothing; a backslash makes a literal `#` or `!`; trailing spaces are dropped unless escaped.
- "If there is a separator at the beginning or middle (or both) of the pattern, then the pattern is relative to the directory level of the particular .gitignore file itself. Otherwise the pattern may also match at any level below the .gitignore level."
- "If there is a separator at the end of the pattern then the pattern will only match directories, otherwise the pattern can match both files and directories."
- `*`, `?` and `[...]` never match a slash; `**` is special only as a leading `**/`, a trailing `/**` or a `/**/` in the middle.
- "a backslash at the end of a pattern is an invalid pattern that never matches."
- "within one level of precedence, the last matching pattern decides the outcome", and "It is not possible to re-include a file if a parent directory of that file is excluded."

Not modelled on the git side: nested `.gitignore` files, `info/exclude`, `core.excludesFile` and `core.ignorecase`. Files git already tracks are not affected by `.gitignore` at all.

**Docker**, implemented from the [build context documentation](https://docs.docker.com/build/concepts/context/) and from the source of [moby/patternmatcher](https://github.com/moby/patternmatcher) (`ignorefile/ignorefile.go`, `patternmatcher.go`) and [tonistiigi/fsutil](https://github.com/tonistiigi/fsutil) (`filter.go`), read at patternmatcher commit `5a6d8429a19bb6948a372ff19e86fe83599a04b7` and fsutil commit `c1c5bffae3b59e780de4d832ef10a508d93c37cb`:

- The docs say "Matching is done using Go's filepath.Match function rules. A preprocessing step uses Go's filepath.Clean function to trim whitespace and remove . and .. ." and "Leading and trailing slashes in ignore patterns are disregarded." The patternmatcher reader states: "Leading forward-slashes ("/") are removed from ignore patterns, so "/some/path" and "some/path" are considered equivalent."
- "Beyond Go's filepath.Match rules, Docker also supports a special wildcard string ** that matches any number of directories (including zero)."
- Each pattern is compiled into a regular expression anchored at both ends, so there is no implicit `**/` in front. The docs' own example: "All markdown files right under the context directory except README.md are excluded from the context. Note that markdown files under subdirectories are still included."
- "the last line of the .dockerignore that matches a particular file determines whether it's included or excluded."
- A `!` line is tested against the file itself. fsutil's walk only skips an excluded directory when no exception could reach inside it; its source comment reads "Optimization: we can skip walking this dir if no exceptions to exclude patterns could match anything inside it." That is why a `!` line naming a path inside an excluded directory re-includes it in Docker, while git's documentation says it cannot.
- "For historical reasons, the pattern . is ignored."
- A lone `!` is rejected with patternmatcher's error `illegal exclusion pattern: "!"`, and a pattern Go's filepath.Match calls malformed (an unclosed `[`, a trailing backslash) fails the build. The page shows "Docker would reject this file" instead of guessing.
- "A Dockerfile-specific ignore-file takes precedence over the .dockerignore file at the root of the build context if both exist." The specific file sits next to the Dockerfile and is named after it, for example `docker/build.Dockerfile.dockerignore`.
- An excluded Dockerfile or `.dockerignore` gets a note: "These files are still sent to the builder as they're needed for running the build. But you can't copy the files into the image using ADD , COPY , or bind mounts."

- patternmatcher's escape table (`escapeBytes`, an `[8]byte` indexed by `b%8` with bit `b/8`) only reaches characters below code 64, so `|`, `{` and `}` in a wildcard line reach Go's regular expression engine unescaped. With patternmatcher v0.6.1, the version Docker CLI 29.8.2 and buildx v0.37.2 depend on, the line `[ab]|` matches every path and `?a{2}` matches `xaa`. The page follows that reading and flags the line.

The Docker column models the client side walk on Linux or macOS, where a backslash escapes the next character. On a Windows client patternmatcher treats a backslash as a path separator; that case is not modelled. A pattern escape such as `\d`, which Go's regular expression engine reads as a character class, is read that way and flagged when it appears; other letter escapes are not modelled. The Go regular expression is matched by the page's own matcher, which runs in time proportional to the pattern times the path, as Go's engine does, so no line can stall the page.

## Verification

Run on 2026-10-16 against the worked example above, in a scratch directory holding exactly its eleven paths:

- git column: `git check-ignore -v --no-index -n` with git version 2.55.0.windows.5. Every row matched, deciding line included.
- Docker column: a real build of the context with the file above as `.dockerignore` and this Dockerfile:

  ```
  FROM busybox
  COPY . /ctx
  RUN find /ctx -type f | sort
  ```

  Docker client 29.8.2 (windows/amd64), Docker Engine 29.8.2 (linux/amd64), buildx v0.37.2. The listed files were `.dockerignore`, `Dockerfile`, `README.md`, `docs/README.md`, `docs/x.md`, `logs/build.log`, `node_modules/keep.txt` and `src/target/b.txt`, which is exactly the set the Docker column marks as sent.
- With the same Docker versions, a `.dockerignore` holding only `!` failed with `illegal exclusion pattern: "!"`, and one holding only `[a` or only `abc\` failed with `syntax error in pattern`. A file holding `x` then `!a*[` failed with `failed to match excludepatterns: syntax error in pattern` once a file named `x` was in the context, and built normally when only `!a*[` was present. The page reports each of these the same way.

The page runs its self test on load and prints a live pass and fail count. It covers the worked example in both columns, at least one fixture per rule G1 to G7 and D1 to D8, the Docker docs' `*/temp*`, `*/*/temp*` and `temp?` example and its two `README*.md` ordering examples rewritten with other file names, and a positive control written to fail.

## Why this exists

The same text file is read by two tools that each follow their own published rules. Nobody has a bug to file; the failure only appears when one file is reused as the other, and it is visible only with both readings side by side. The question keeps coming up in public trackers, for example moby/moby#40318 "Verify parity between .dockerignore and .gitignore syntax", mlflow/mlflow#21151 ".dockerignore patterns don't exclude nested directories" and testcontainers/testcontainers-dotnet#1768 "fix: Prevent Docker ignore patterns from matching path prefixes".

Single file, no build step, no network call, no tracking, MIT licensed. Paste a real ignore file and a real file list without wondering where they go.

## Privacy

Everything runs in your browser. The page makes no network call of any kind: no code, font, analytics or telemetry request, and nothing you paste is transmitted. A Content-Security-Policy in the page forbids network connections. The only thing it stores is your light or dark theme choice, in your browser's local storage. Save the file and it works offline from `file://`.

## Run locally

```
git clone https://github.com/0xelitesystem/dockerignore-vs-gitignore-pattern-tester.git
cd dockerignore-vs-gitignore-pattern-tester
```

Open `index.html` in a browser. Or serve it:

```
python -m http.server 8000
```

then open `http://localhost:8000/`.

## Build

No build. One self-contained `index.html` with inline CSS and JavaScript. No dependencies, no package manager, no bundler, no CDN, no web font.

## Independence and provenance

This is an independent tool. It is not affiliated with or endorsed by Docker, Inc., the Git project or the Go project. The git column was implemented from gitignore(5). Parts of the Docker column are JavaScript translations of the Go source named below; each one is listed under Third-party notices, and the upstream licence texts are reproduced in a comment at the top of the script in `index.html`, which also states what was changed.

## Third-party notices

**Go standard library, `path/filepath`** (https://github.com/golang/go). Licensed BSD-3-Clause. Holder line, copied from its LICENSE file: "Copyright 2009 The Go Authors." What was taken: the `filepath.Match` algorithm in `src/path/filepath/match.go` (`Match`, `scanChunk`, `matchChunk`, `getEsc`), translated to JavaScript as the page's syntax check, `goMatchSyntax`.

**moby/patternmatcher** (https://github.com/moby/patternmatcher). Licensed Apache-2.0. Holder lines: "Copyright 2013-2018 Docker, Inc." from its LICENSE file, and "Copyright 2012-2017 Docker, Inc." from its NOTICE file. What was taken: `ReadAll` (`ignorefile/ignorefile.go`) and `Pattern.compile`, `Pattern.match` and `PatternMatcher.MatchesUsingParentResults` (`patternmatcher.go`), read at commit `5a6d8429a19bb6948a372ff19e86fe83599a04b7`, translated to JavaScript and adapted to a pasted path list (`parseDocker`, `dockerCompile`, `dockerPatMatch`, `dockerMatchNode`). Two short strings are also quoted: the reader's doc comment on leading slashes and the error text `illegal exclusion pattern: "!"`. The NOTICE text and the Apache License 2.0 text are reproduced in `index.html`.

**tonistiigi/fsutil** (https://github.com/tonistiigi/fsutil). Licensed MIT. Holder line, copied from its LICENSE file: "Copyright 2017 Tõnis Tiigi <tonistiigi@gmail.com>". What was taken: the directory skip logic of `filter.go` (`patternWithoutTrailingGlob`, the `onlyPrefixExcludeExceptions` test in `NewFilterFS` and the skip in `filterFS.Walk`), read at commit `c1c5bffae3b59e780de4d832ef10a508d93c37cb`, translated to JavaScript (`withoutTrailingGlob`, `parseDocker`, `dockerEval`), and one source comment quoted above. The MIT licence text is reproduced in `index.html`.

**Docker documentation** (https://docs.docker.com/build/concepts/context/), from the docker/docs repository, licensed Apache-2.0. What was taken: the short sentences quoted on the page and in this README, with attribution.

**gitignore(5)** (https://git-scm.com/docs/gitignore). What was taken: the short sentences quoted on the page and in this README, with attribution. No git code is reproduced.

## Related

- https://github.com/0xelitesystem/dockerfile-linter
- https://github.com/0xelitesystem/docker-basics-reference
- https://github.com/0xelitesystem/git-basics-reference

## License

MIT. Copyright (c) 2026 0xelitesystem. See [LICENSE](LICENSE).
