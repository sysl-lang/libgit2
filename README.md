# libgit2

[libgit2](https://libgit2.org/), git as a library, bound for sysl. It covers repositories (init,
open, discover, local clone), the index, commits, trees, blobs and tree building, references,
branches and tags, checkout, history walks, diffs with patch text, status, blame, configuration, the
stash, and credentials. Callbacks for checkout progress, fetch progress, diff walks, status walks,
tag and stash walks, and credentials are ordinary sysl closures.

*An independent sysl package for libgit2. It is not affiliated with or endorsed by libgit2's authors or trademark holders.*

```hocon
dependencies {
  libgit2 { git = "github.com/sysl-lang/libgit2", version = "0.1.0" }
}
```

```sysl
import sh.sysl.libgit2.*
```

libgit2 1.9 or newer has to be installed with its headers (`brew install libgit2`; Debian's
`libgit2-dev`). The build asks pkg-config for `libgit2`. libgit2 is GPL-2.0 with a linking exception;
this package is ISC.

## An example

This commits files onto `HEAD` and then prints the log. It is `example_tests.sysl`, compiled and run by
`sysl test .`:

```sysl
/** Commit `files` onto `HEAD` of the repository at `dir`, then answer `git log --oneline`. */
commit_and_log(dir: string, files: []const (string, string), message: string) -> Result[string, Error]
    val repo = open(dir)?
    val index = repo.index()?

    for (name, text) in files
        write_text(s"$dir/$name", text).expect(s"wrote $name")
        index.add(name)?

    index.write()?

    val tree = repo.find_tree(index.write_tree()?)?
    val me = signature_now("Ann Author", "ann@example.com")?
    val parents: Buf[&Commit] = buf()

    if !repo.head_unborn()? then parents.push(repo.revparse("HEAD")?.as_commit()?)

    repo.commit(Some("HEAD"), me, me, message, tree, parents.view())?

    val walk = repo.revwalk()?

    walk.push_head()?

    var log = ""

    for id in walk.ids()?
        log += s"${id.short()} ${repo.find_commit(id)?.summary()}\n"
    Ok(log)
```

## How it is shaped

**Two layers.** `sh.sysl.libgit2.c` declares the C API as it is. `sh.sysl.libgit2` is what a program
imports. Every libgit2 object there is a `&T` whose `Drop` calls the matching `git_*_free` exactly
once.

**Everything read out of a repository keeps the repository open.** A commit, tree, blob, tag,
reference, index, diff or walk holds a share of the `&Repository` it came from, because libgit2
requires a repository to outlive every object it handed out. A tree entry holds its `&Tree` the same
way. Signatures, deltas, hunks, lines, blame hunks, statuses and stash entries are copied out into
plain sysl values, so they borrow nothing.

**Errors.** A failing call answers `Err(Error)`. The error carries libgit2's return code as a `Code`
(`NotFound`, `UnbornBranch`, `Conflict`, ...), the class `git_error_last()` reported as a `Class`
(`Reference`, `Checkout`, ...), and libgit2's own message, for example:

| call | `code` | `class` | `message` |
|---|---|---|---|
| `open` on a directory that is not a repository | `NotFound` | `Repository` | `could not find repository at '<dir>'` |
| `find_reference("refs/heads/nope")` | `NotFound` | `Reference` | `reference 'refs/heads/nope' not found` |
| `head()` on an unborn branch | `UnbornBranch` | `Reference` | `reference 'refs/heads/main' not found` |
| `checkout_branch` over an uncommitted change | `Conflict` | `Checkout` | `1 conflict prevents checkout` |

**Callbacks.** Each callback is a closure boxed beside the call. libgit2's `void *payload` points at
the box, and a named sysl function recovers it with `ptr_cast`. All of these walks run on the calling
thread, so the closures are plain `&Fn`, not `&sync Fn`. A walk callback answers `false` to stop, and
a stop counts as success, not as an error.

**Credentials are bound but not exercised over a network.** `clone(..., credentials = Some(...))`
installs the trampoline. libgit2 asks it only when an HTTPS or SSH remote wants credentials, and a
local clone never does. The tests have no network, so they call the trampoline directly with the
payload a clone would pass. They check that the request arrives as asked (URL, user name, allowed
kinds), that a `Password` answer becomes a real `git_credential` with that user name, and that no
closure, or an answer of `None`, passes through as `GIT_PASSTHROUGH`.

**The user's configuration.** A repository's config is layered with `~/.gitconfig` and the system
file. `set_search_path(Global, dir)` (`GIT_OPT_SET_SEARCH_PATH`) moves those levels elsewhere, which
is how the tests shut them out.

**A blob's raw bytes are borrowed.** `Blob.content()` answers a `Content`: the slice into libgit2's
buffer, together with the `&Blob` that keeps the buffer alive. A slice cannot yet keep a C owner alive
by itself, so the slice must not be kept longer than the `Content` it came in. `Blob.text()` copies
the bytes instead.

## The survey: what is bound, and what is deliberately absent

libgit2 1.9.7 exports 950 functions across 60 public headers plus `sys/`. Classified against what
sysl can express:

- **Expressible directly: all of it.** Opaque handles, out-parameters, `git_oid` (20 bytes), and
  `git_buf` and `git_strarray` by pointer are all expressible. So are callbacks with a `void *`
  payload, through `&f`. Option structs (checkout, status, diff, blame, stash apply) are declared
  field by field, and their sizes and key offsets are asserted at compile time against the header
  (`@assert` in `c/c.sysl`). `git_clone_options` nests about forty fields, so it is held as storage
  the header sizes, and the five fields this binding sets are written at `offsetof` positions. The
  variadic `git_libgit2_opts` is declared as variadic.
- **Shim-able: none needed.** No line of C was written: no `static inline`, union or bitfield stands
  in the way of what is bound.
- **Needs the language to move: one known gap**, already decided for alpha.13 (below).

**Bound (about 170 functions):** library init and options (search path); repository open, open_ext,
init, discover, clone, state, cleanup, head, unborn, detached, is_empty, is_bare, path, workdir,
set_head, set_head_detached, index, config; oids; objects (lookup, peel, revparse); commits (read and
create); trees and tree entries; tree builder; blobs (create, read, binary check); the index (add,
add_all, remove, read, write, write_tree, conflicts); references (lookup, create, symbolic, resolve,
list, delete, name_to_id); branches (create, lookup, iterate, delete, is_head, name); tags (annotated,
lightweight, lookup, list, foreach, delete); checkout (tree, head, branch) with progress; revwalk
(push, push_head, push_ref, hide, sorting; an `Iterate` cursor); diffs (tree to tree, tree to index,
index to workdir, untracked) with deltas, whole and per-file patch text, and a file/hunk/line walk;
status (walk, list, single file, ignored); blame; configuration (repository and on-disk file, string,
int, bool, remove); the stash (save, apply, pop, drop, walk); credentials (password, ssh-agent,
default).

**Deliberately absent, and why:**

- **Network remotes** (fetch, push, `git_remote_*`, transports, certificate checks, proxies). The
  closure machinery for them is the one clone already uses, but nothing can test them without a
  network or a server. Binding them untested would break the org's rule.
- **Merge, rebase, cherry-pick, revert, apply, describe, notes, submodules, worktrees, the reflog,
  mailmap, attributes, filters, pathspec, the ODB and refdb backends, packbuilder, indexer and
  `sys/`.** These were left out to keep the first release reviewable, not for a language reason. The
  survey found nothing in them that sysl cannot express, and each follows a pattern already shown
  here.
- **`deprecated.h`** (68 functions), which are only old names for bound calls.
- **SHA-256 object ids.** Homebrew's libgit2 is built without `GIT_EXPERIMENTAL_SHA256`, and the
  compile-time size assertion on `git_oid` would fail on a build that has it.

## Gaps found in sysl

- **A slice over memory C owns cannot keep the C owner alive** (already decided for alpha.13).
  `Blob.content()` returns the slice beside the `&Blob` because it cannot make the slice carry it.

## Tests

`sysl test .` runs 46 tests. Each is checked against the `git` command line, run on the same repository
in a scratch directory with no configuration of its own:

- commits the binding writes against `git cat-file -p`, byte for byte;
- repositories `git` made, read by the binding (head, commit, signatures, tree against `git ls-tree`,
  blob);
- blob ids against `git hash-object`, and built trees against `git ls-tree`;
- walks against `git log` (including `A..` and `--reverse`);
- branches, references and tags against `git branch`, `for-each-ref`, `cat-file` and `tag -l`;
- patch text against `git diff`, `git diff A B`, `git diff -U1`/`-U0` and `git diff --cached`, byte
  for byte;
- status against `git status --porcelain -uall [--ignored]`;
- blame against `git blame --porcelain`;
- the stash against `git stash list`, and config against `git config [--local | -f]`;
- local clones (plain, bare and by branch) against `git log`, `remote get-url` and `rev-parse`.

The error paths are checked with their exact code, class and message: a non-repository, an unknown
reference and revision, an unborn `HEAD`, a conflicting checkout, a duplicate branch, a missing
status path, a missing blame path, an empty stash, a missing config key, a bad integer, a clone into
a non-empty directory, a clone from a missing source, and short or bad object ids.

Under AddressSanitizer: `SYSL_EXTRA_CFLAGS="-fsanitize=address -g" sysl test .`. This is a
`pkg_config` package, so **only the sysl half is instrumented**. libgit2 is Homebrew's
uninstrumented build. ASan still replaces `malloc` and `free` for the whole process, so a
double-freed libgit2 object would be caught.

A mutation pass breaks each parameter the binding passes on: checkout strategy, the status and diff
flags, context lines, signature offset, committer, blame's newest commit, stash flags, bare, branch,
force, reverse, bool encoding, tag message, credential fields, and error-code mapping. Every one of
them turns at least one test red.

## License

ISC. See `LICENSE`.
