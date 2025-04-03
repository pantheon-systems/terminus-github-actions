# Terminus Multidev Manager Action

This GitHub Action helps manage Pantheon multidev environments by ensuring there are enough available slots for new environments. If needed, it will automatically remove the oldest environments to make room for new ones.

## Inputs

### `pantheon-site-name`
**Required** The name of your Pantheon site.

### `multidev-count`
**Required** The number of multidev environments you need to ensure are available. Default: "1"

### `pantheon-machine-token`
**Required** Your Pantheon Terminus machine token for authentication.

### `protected-environments`
**Optional** Comma-separated list of environment names that should never be deleted. Default: "dev,test,live"

## Example Usage

```yaml

steps:
  - name: Setup PHP
    uses: shivammathur/setup-php@v2
    with:
      php-version: "8.2"

  - name: Install Terminus
    uses: pantheon-systems/terminus-github-actions@v1
    with:
      pantheon-machine-token: ${{ secrets.PANTHEON_MACHINE_TOKEN }}

  - uses: pantheon-systems/terminus-github-actions/ensure-multidev@v1
    with:
      pantheon-site-name: 'my-pantheon-site'
      multidev-count: '2'
      pantheon-machine-token: ${{ secrets.PANTHEON_MACHINE_TOKEN }}
      protected-environments: 'dev,test,live,some-multidev-environment-thats-important'

```

## How it works

1. The action authenticates with Terminus using the provided token
2. Checks the current number of multidev environments
3. If there isn't enough room for the requested number of new environments:
- Lists all existing multidev environments
- Filters out protected environments
- Sorts by creation date
- Deletes the oldest environments until there's enough room

## Important Notes

- The action will never delete the `dev`, `test`, or `live` environments
- Protected environments specified in `protected-environments` will never be deleted
- The action will fail if it cannot free up enough environments after deleting all eligible ones