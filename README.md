# GitHub Actions Cache Key Check

This GitHub Action determines if for a given key or prefix a cache restore would have caused a cache hit under the current conditions without restoring the cache contents itself.

## Goal
This action can be useful if you want to determine if you should preload a cache so you can use it in a different job later. Especially if that job is a matrix. If the cache is present, you can just use it your matrix later on. If it isn't, you can preload the cache and use it in your matrix without generating `n` cache misses (where `n` is the number of jobs in your matrix) which all try to save the cache under the same key.

An alternative would be to use the [actions/cache/restore](https://github.com/actions/cache/blob/main/restore/README.md) action to restore the cache entry and if it returns a cache-hit, just ignore it. However this would be a waste of bandwith most of the times.

In a linear workflow this is not that useful as you can just restore the cache, generate it if cache-miss has occured and use it afterwards.

## Implementation
GitHub caches have [restrictions](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows#restrictions-for-accessing-a-cache) on which caches it can restore depending on what branch or tag it is used in. This action uses the GitHub API to get access to the list of available caches and parses the output to determine if cache-hit would occur adhering to these restrictions.

To keep it simple, this is just a composite action. As composite actions cannot use types (`int`, `boolean` etc.) on outputs it always returns a `string`. Keep this in mind when parsing the output!

Currently there is a [outstanding pull request](https://github.com/actions/cache/pull/1041) to implement this natively into `actions/cache`.

When using this action, be aware of the [TOCTOU](https://en.wikipedia.org/wiki/Time-of-check_to_time-of-use) problem. It is recommended to not rely entirely on the output of the this check and make sure your workflow can also still succeed (although less efficient) if a false cache-hit has occured.

## Usage

### Pre-requisites
This actions uses the GitHub CLI to do the API requests. Therefore the GitHub CLI must be present on your runner. You also need valid token with `actions:read` access. Usually this is just your `GITHUB_TOKEN`, but you can also create a [personal access token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token). You can either set `GH_TOKEN` in your entire environment (so all steps can use it) or pass the key to the `repo-token` input.

### Inputs
* `key` (not required) - An explicit key or prefix for identifying the cache
* `repo-token` (not required) - Token to use to authorize actions API access. Typically the GITHUB_TOKEN secret, with `actions:read` access

### Outputs
* `cache-hit` - A string ('true' or 'false') to indicate a match was found for the key or prefix

### Examples

#### Token set in environment
```yml
name: CI
on: push

jobs:
  setup:
    runs-on: ubuntu-latest
    permissions:
      # Default
      contents: read
      packages: read
      # Custom, for API cache access
      actions: read
    env:
      GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    steps:
      - name: Check for cache hit
        uses: pascallj/cache-key-check@v1
        id: cache-primes
        with:
          key: primes

      - name: Generate Prime Numbers
        if: steps.cache-primes.outputs.cache-hit != 'true'
        run: /generate-primes.sh -d prime-numbers

      - uses: actions/cache/save@v3
        if: steps.cache-primes.outputs.cache-hit != 'true'
        with:
          path: prime-numbers
          key: primes

  tests:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        shell: [sh, dash, bash]
    steps:
      - uses: actions/cache@v3
        with:
          path: prime-numbers
          key: primes

      - name: Use Prime Numbers
        run: ${{ matrix.shell }} -c /primes.sh -d prime-numbers
```
#### Token passed to action
```yml
name: CI
on: push

jobs:
  setup:
    runs-on: ubuntu-latest
    steps:
      - name: Check for cache hit
        uses: pascallj/cache-key-check@v1
        id: cache-primes
        with:
          key: primes
          repo-token: ${{ secrets.CUSTOM_GITHUB_TOKEN }}

      - name: Generate Prime Numbers
        if: steps.cache-primes.outputs.cache-hit != 'true'
        run: /generate-primes.sh -d prime-numbers

      - uses: actions/cache/save@v3
        if: steps.cache-primes.outputs.cache-hit != 'true'
        with:
          path: prime-numbers
          key: primes

  tests:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        shell: [sh, dash, bash]
    steps:
      - uses: actions/cache@v3
        with:
          path: prime-numbers
          key: primes

      - name: Use Prime Numbers
        run: ${{ matrix.shell }} -c /primes.sh -d prime-numbers
```
