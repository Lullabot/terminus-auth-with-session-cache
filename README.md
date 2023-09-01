# GitHub Action for caching Terminus session

Caching and reusing the authenticated Terminus session between jobs helps
prevent login errors due to Auth0 rate limiting during parallel jobs all trying
to authenticate at the same time.

## DDEV
If you use [DDEV](https://ddev.com/) in GitHub Actions, you can pass "`ddev:
true`" to authenticate Terminus inside the DDEV container. Before calling this
action with "`ddev: true`", install and start DDEV with one of these actions:

- Official: [Setup DDEV in Github Workflows](https://github.com/marketplace/actions/setup-ddev-in-github-workflows)
- [Drainpipe](https://github.com/Lullabot/drainpipe)'s version has additional settings for configuring SSH keys, git user, and composer cache directory: [action.yml](https://github.com/Lullabot/drainpipe/blob/main/scaffold/github/actions/common/ddev/action.yml)

## Example: Caching a session token before matrix'd jobs
Make sure a Terminus session is stored in cache before attempting to run jobs in
parallel. We can accomplish this by adding a "needs" dependency on a preliminary
job that refreshes the Terminus session token:
```yaml
name: Example workflow demonstrating how to authenticate Terminus before
  dispatching jobs in parallel
on:
  push:
    branches:
      - 'main'
jobs:
  TerminusGetSession:
    runs-on: ubuntu-22.04
    # Always store production secrets in a GitHub "Environment" that restricts
    # which branches can deploy to it, and protect those branches with rules
    # including code review and security tests.
    # @see https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment
    environment: Pantheon
    steps:
      - name: Populate a new Terminus session in cache before dispatching the
          matrix'd job
        uses: lullabot/terminus-auth-with-session-cache@v2
        with:
          pantheon-machine-token: ${{ secrets.PANTHEON_MACHINE_TOKEN }}
          # Note: DDEV is never needed for this step since we're just trying to
          # generate and cache the session token for later jobs (which may use
          # DDEV).
          # ddev: false
  ExampleParallelJob:
    # This "needs" dependency makes sure the session is cached before starting
    # these parallel jobs.
    needs: [TerminusGetSession]
    runs-on: ubuntu-22.04
    environment: Pantheon
    strategy:
      matrix: ['site-a', 'site-b', 'site-c']
    steps:
      - name: Log in to Terminus by restoring existing session or initiating new
        uses: lullabot/terminus-auth-with-session-cache@v2
        with:
          pantheon-machine-token: ${{ secrets.PANTHEON_MACHINE_TOKEN }}
          # Set 'ddev: true' if using DDEV in GitHub Actions.
          ddev: false
      - name: Example step, clear caches on each site
        run: ddev terminus drush ${{ matrix.site-name }}.live -- cache:rebuild
        shell: bash
```

## Example: Refresh Terminus sessions during off-peak hours
If you have many workflows running simultaneously, you may opt for generating
new Terminus sessions during off-peak hours, so an old session doesn't expire at
a random moment when concurrent jobs are running.
```yaml
name: Refresh 24h Dev and Prod Terminus sessions during off-peak hours
on:
  schedule:
    # 8am UTC is 3am US Central Time
    - cron:  '0 8 * * *'
jobs:
  TerminusGetDevSession:
    runs-on: ubuntu-22.04
    steps:
      - name: Refresh the Development Terminus session in cache
        uses: lullabot/terminus-auth-with-session-cache@v2
        with:
          # Development token secret may be repository-wide.
          pantheon-machine-token: ${{ secrets.PANTHEON_MACHINE_TOKEN_DEV }}
          # Tell the action to grab a fresh session and ignore any existing
          # cache.
          force-new-session: true
  TerminusGetProdSession:
    runs-on: ubuntu-22.04
    environment: Pantheon
    steps:
      - name: Refresh the Production Terminus session in cache
        uses: lullabot/terminus-auth-with-session-cache@v2
        with:
          # Production token secret should be protected behind an Environment.
          pantheon-machine-token: ${{ secrets.PANTHEON_MACHINE_TOKEN_PRODUCTION }}
          # Tell the action to grab a fresh session and ignore any existing
          # cache.
          force-new-session: true
```

## Pantheon documentation
The related Pantheon documentation lives here: [Authenticate Terminus in a
GitHub Actions Pipeline](https://docs.pantheon.io/terminus/ci/github-actions).
It currently recommends a different approach but hopefully will be updated soon
to reference this action or an equivalent one.
