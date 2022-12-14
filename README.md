# Template Haskell package repository

This git repository contains a template to create Haskell package
repositories compatible with Cabal.

You will need to create TUF keys for this repository:

```bash
$ nix run github:andreabedini/foliage create-keys
$ KEYS=$(tar cz -C _keys . | base64)
$ gh secret set KEYS --body "$KEYS" # or through the GitHub web interface
```

The public keys are just the name of the key files.

```bash
$ tree _keys/root/
_keys/root/
├── 3d3ea2a09cd9d587a8e774d6cba28cfbd0a29c962273fe5bdb6677aa4771ebf6.json
├── 551b61fe69665e52cf0037bb551f2b064f2af4e2d6bdac08d70bcda21dedc65c.json
└── b645d0896f0257eaab36def75b4b70c85cea8f826a202bdc040da2a409a6a81a.json
```

## What is a Cabal package repository?

A package repository is essentially a mapping from package name and version
to the source distribution for the package. Most Haskell programmers will be
familiar with the package repository hosted on Hackage, which is enabled
by default in Cabal.

However, Cabal supports the use of _additional_ package repositories.
This is convenient for users who can't or don't want to put their packages
on Hackage.

## How to use the package repository

To use the package repository from cabal, add the following lines to your
`cabal.project` file:

```
-- Use a unique name here, otherwise cabal will get confused
repository github-andreabedini-foliage-template
  -- Use the GitHub pages URL for your repository here
  url: https://andreabedini.github.io/foliage-template
  secure: True
  -- Read above how to create keys
  root-keys:
    3d3ea2a09cd9d587a8e774d6cba28cfbd0a29c962273fe5bdb6677aa4771ebf6
    551b61fe69665e52cf0037bb551f2b064f2af4e2d6bdac08d70bcda21dedc65c
    b645d0896f0257eaab36def75b4b70c85cea8f826a202bdc040da2a409a6a81a
```

The package repository will be understood by cabal, and can be updated with
`cabal update`.

The `index-state` for the package repository can also be pinned in as
usual:

```
index-state: foliage-template 2022-08-25T00:00:00Z
```

### ... with haskell.nix

To use the package repository with `haskell.nix`, do the following:

1. Add the package repository to your `cabal.project` as above.
2. Setup a fetcher for the package repository. The easiest way is to use a flake input, such as:
```
inputs.github-andreabedini-foliage-template = {
  url = "github:andreabedini/foliage-template?ref=repo";
  flake = false;
};
```
3. Add the fetched input to the `inputMap` argument of `cabalProject`, like this:
```
cabalProject {
  ...
  inputMap = { "https://andreabedini.github.io/foliage-template" = github-andreabedini-foliage-template; };
}
```

When you want to update the state of the package repository, you can simply
update the flake input (in the example above you would run `nix flake lock
--update-input github-andreabedini-foliage-template`).

If you have the repository configured correctly, then when you run `cabal
build` from inside a `haskell.nix` shell, you should not see any of the
packages in the repository being built by cabal.  The exception is if you
have a `source-repository-package` stanza which overrides a _dependency_ of
one of the packages in the repository. Then cabal will rebuild them both.
If this becomes a problem, you can consider adding the patched package to
the repository itself, see
[below](#how-do-i-add-a-patched-versions-of-a-hackage-package-to-this-package-repository).

## How to add a new package (or package version) to the repository

Package versions are defined using metadata files `_sources/$pkg_name/$pkg_version/meta.toml`,
which you can create directly. The metadata files have the following format:

```toml
# REQUIRED timestamp at which the package appears in the index
timestamp = 2022-03-29T06:19:50+00:00
# REQUIRED URL pointing to the source code tarball (not decessarily a sdist)
url = 'https://github.com/input-output-hk/ouroboros-network/tarball/fa10cb4eef1e7d3e095cec3c2bb1210774b7e5fa'
# OPTIONAL subdirectory inside the tarball where the package is located
subdir = 'typed-protocols'
```

NOTE: When adding a package, it is important to use a timestamp that is
greater than any other timestamp in the index. Indeed, cabal users rely on
the changes to the repository index to be append-only. A non append-only
change to the package index would change the repository index state as
pinned by `index-state`, breaking reproducibility.

Using the current date and time (e.g. `date --utc +%Y-%m-%dT%H:%M:%SZ`)
works alright but if you are sending a PR you need to consider the
possibility that another developer has inserted a new (greater) timestamp
before your PR got merged. We have CI check that prevents this from
happening, and we enforce FF-only merges.

### ... from GitHub

There is a convenience script `./scripts/add-from-github.sh` to simplify
adding a package from a GitHub repository.

```console
$ ./scripts/add-from-github.sh
Usage: ./scripts/add-from-github.sh REPO_URL REV [SUBDIR...]
```

The script will:

1. Find the cabal files in the repo (either at the root or in the specified
   subdirectories)
2. Obtain package names and versions from the cabal files
3. Create the corresponding `meta.toml` files
4. Commit the changes to the repository

## How do I add a patched versions of a Hackage package to this package repository?

This package repository should mostly contain versions of packages which are _not_ on Hackage.

If you need to patch a version of a package on Hackage, then there are two options:

1. For short lived forks, use a `source-repository-package` stanza by
   preference.
2. For long-lived forks (because e.g. the maintainer is unresponsive or the
   patch is large and will take time to upstream), then we can consider
   releasing a patched version in this package repository.

The main constraint when adding a patched version to this repository is to
be sure that we use a version number that won't ever conflict with a
release made by upstream on Hackage.  There are two approaches to doing
this:

1. Release the package into this package repository under a different name
   (for the fork).  This is very safe, but may not be possible if the
   dependency is incurred via a packge we don't control, as then we can't
   force it to depend on the renamed package.
2. Release the package under a version that is very unlikely to be used by
   upstream.  The scheme that we typically use is to take the existing
   version number, add four zero components and then a patch version, e.g.
   `1.2.3.4.0.0.0.0.1`.

## How to test changes to this package repository against haskell.nix projects

Sometimes it is useful to test in advance how a new package or a cabal file
revision affect a certain project. If the project uses haskell.nix, one can
follow these steps:

- Make a local checkout of this repository and make the intended changes
- Build the repository with `nix shell -c foliage build`
- Build the project to test overriding the repository with your local
  version in `_repo`.
  ```bash
  $ nix build \
    --override-input github-andreabedini-foliage-template path:/somewhere/foliage-template/_repo
  ```
- In particular you can examine the build plan without completing the
  build:
  ```bash
  $ nix build .#default.project.plan-nix.json \
    --out-link plan.json                      \
    --override-input github-andreabedini-foliage-template path:/somewhere/foliage-template/_repo
  ```
  (adjust `.#default` to your flake).
- Note that you might need to bump the index-state to allow cabal to see
  the changes in the repository.

## CI for this repository

The CI for this repository does the following things:

- Checks that the timestamps in the repository are monotonically increasing
  through commits. Along with requiring linear history, this ensures that
  repository that we build is always an extension of the previous one.
- Builds the package repository from the metadata using `foliage`.
- Deploys the package repository to the `repo` branch, along with some
  static web content. The branch will be created if it doesn't already
  exists.

## Help!

If you have trouble, open an issue, or contact the maintainers:

- Andrea Bedini (andrea@andreabedini.com)
