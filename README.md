# Setup Terminus

A GitHub Action for quickly installing and configuring the Pantheon CLI tool,
[Terminus](https://github.com/pantheon-systems/terminus).

## Usage

In order to avoid deprecation warnings, it's recommended to use the
[`setup-php`](https://github.com/shivammathur/setup-php) action rather than rely
on the version of PHP that is installed by default on GH runners.

```yaml
steps:
  - name: Setup PHP
    uses: shivammathur/setup-php@v2
    with:
      php-version: '7.4'

  - name: Install Terminus
    uses: ackama/setup-terminus@main
    with:
      pantheon-machine-token: ${{ secrets.PANTHEON_MACHINE_TOKEN }}

  - name: List sites
    runs: terminus site:list
```
