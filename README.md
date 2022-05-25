[![Actively Maintained](https://img.shields.io/badge/Pantheon-Actively_Maintained-yellow?logo=pantheon&color=FFDC28)](https://pantheon.io/docs/oss-support-levels#actively-maintained)

# Terminus GitHub Actions

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
    uses: pantheon-systems/terminus-github-actions@main
    with:
      pantheon-machine-token: ${{ secrets.PANTHEON_MACHINE_TOKEN }}

  - name: List sites
    runs: terminus site:list
```

By default, this action installs the latest version of Terminus that has been
released on GitHub. You can provide a specific version of Terminus to install
using the `terminus-version` input:

```yaml
steps:
  - name: Setup PHP
    uses: shivammathur/setup-php@v2
    with:
      php-version: '7.4'

  - name: Install Terminus
    uses: pantheon-systems/terminus-github-actions@main
    with:
      pantheon-machine-token: ${{ secrets.PANTHEON_MACHINE_TOKEN }}
      terminus-version: 2.6.5

  - name: List sites
    runs: terminus site:list
```

## Credits

Big thanks to <a href="https://github.com/G-Rath">Gareth Jones</a> and <a href="https://www.ackama.com/">Ackama</a> for the initial development work.