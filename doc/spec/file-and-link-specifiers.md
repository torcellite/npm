# `file:` and `link:` specifiers

`specifier` refers to the value part of the `package.json`'s `dependencies`
object.  This is a semver expression for registry dependencies and
URLs and URL-like strings for other types.

### Dependency Specifiers

* A valid `link:` specifier is either an absolute path, (eg
  `/path/to/thing`, `d:\path\to\thing`) or a relative path, eg
  `../path/to/thing`, `path\to\subdir`).  `link:` specifiers do not
  distinguish between forward and backslashes.  That is, we consider `link:a\b`
  and `link:a/b` to be the same value.
* Attempting to install a specifer that has a windows drive letter will
  produce an error on non-Windows systems.

* A valid `file:` specifier points is:
  * a valid package file. That is, a `.tar`, `.tar.gz` or `.tgz` containing
  `<dir>/package.json`.
  * OR, a directory that contains a `package.json`
* And a valid `file:` specifier also is:
  * A plain path, eg `/path/to/thing`, `../foo/bar/baz`, or even
    `path/to/local/thing.tar.gz`
  * A relative `file:relative/path`-type URL.
  * An absolute `file:///absolute/path` with any number of leading slashes
    being treated as a single slash. That is, `file:/foo/bar` and
    `file:///foo/bar` reference the same package.

Relative specifiers are relative to the file they were found in, or, if
provided on the command line, the CWD that the command was run from.

An absolute specifier found in a `package.json` or `npm-shrinkwrap.json` should
warn.

A specifier provided as a command line argument that is on a different drive
is an error.

### Specifier Disambiguation

On the command line, plain paths are allowed.  These paths can be ambiguous
as they could be a path, a plain package name or a github shortcut.  This
ambiguity is resolved by checking to see if either a directory exists that
contains a `package.json`.  If either is the case then the specifier is a
link specifier, otherwise it's a registry or github specifier.

Historically, these ambiguous specifiers were also allowed in the
`package.json`.  Starting in `npm@5` using an ambiguous specifier in your
shrinkwrap will be depricated and will warn.  In `npm@6` it will be an
error.

### Specifier Matching

A specifier is considered to match a dependency on disk when the `realpath`
of the fully resolved specifier matches the `realpath` of the package on disk.

### Saving File Specifiers

When saving to both `package.json` and `npm-shrinkwrap.json` they will be
saved using the `link:../relative/path` form, and the relative path will be
relative to the project's root folder.  This is particularly important to
note for the `npm-shrinkwrap.json` as it means the specifier there will
be different then the original `package.json` (where it was relative to that
`package.json`).

When shrinkwrapping file specifiers, the contents of the destination
package's `node_modules` WILL NOT be included in the shrinkwrap. If you want to lock
down the destination package's `node_modules` you should create a shrinkwrap for it
separately.

This is necessary to support the mono repo use case where many projects link
to the same package.  If each project included its own npm-shrinkwrap.json
then they would each have their own distinct set of transitive dependencies
and they'd step on each other any time you ran an install in one or the other.

NOTE: This should not have an effect on shrinkwrapping of other sorts of
shrinkwrapped packages.

### Installation

File-type specifiers will necessarily not do anything for `fetch` and
`extract` phases.

The symlink should be created during the `finalize` phase

The `preinstall` for file-type specifiers MUST be run AFTER the
`finalize` phase as the symlink may be a relative path reaching outside the
current project root and a symlink that resolves in `.staging` won't resolve
in the package's final resting place.

Dependencies of the symlink will be installed inside the destination folder. 
Hoisting of dependencies outside of a package included this way is not
possible due to Node.js module resolution semantics.

### Removal

Removal should remove the symlink.

Removal MUST NOT remove the transitive dependencies.

### Listing

In listings they should not include a version as the version is not
something `npm` is concerned about.  This also makes them easily
distinguishable from symlinks of packages that have other dependency
specifiers.

If you had run:

```
npm install --save link:../a
```

And then run:
```
npm ls
```

You would see:

```
example-package@1.0.0 /path/to/example-package
└── a → link:../a
```

```
example-package@1.0.0 /path/to/example-package
+-- a -> link:../a
```

Of note here: No version is included as the relavent detail is WHERE the
package came from, not what version happened to be in that path.

If we had to fallback to copying then instead you will see:

```
example-package@1.0.0 /path/to/example-package
└── a → Copied from: link:../a
```

```
example-package@1.0.0 /path/to/example-package
+-- a -> Copied from: link:../a
```

### Outdated

Local specifiers should only show up in `npm outdated` if they're missing
and when they do, they should be reported as:

```
Package  Current  Wanted  Latest  Location
a        MISSING   LOCAL   LOCAL  example-package
```

### Updating

If a dependency with a local specifier is already installed then `npm
update` shouldn't do anything.  If one is missing then it should be
installed as if you ran `npm install`.
