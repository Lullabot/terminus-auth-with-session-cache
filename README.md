# GitHub Action for caching Terminus session

Caching and reusing the authenticated session token helps prevent login rate
limiting during parallel jobs.

## Example of caching session token before matrix'd job
Make sure a Terminus session is stored in cache before attempting to run jobs in
parallel.
```yaml
name: Example workflow logging in to Terminus before calling parallel jobs
on: # ...
jobs:
  TerminusGetProdSession:
    runs-on: ubuntu-22.04
    # Protect your production secrets with an environment that has branch
    # restrictions.
    environment: Pantheon
    steps:
      - name: Populate a new PROD Terminus session in cache before dispatching
        th matrix'd job
        uses: lullabot/terminus-auth-with-session-cache@v2
        with:
          pantheon-machine-token: ${{ secrets.PANTHEON_MACHINE_TOKEN_PRODUCTION }}
          # If you don't have a cron job to refresh tokens daily, consider
          # setting 'force-new-session: true' here to always authenticate a new
          # session, so it doesn't expire while running the dependent jobs.
          force-new-session: false
          # DDEV is never needed for this step.
          ddev: false
  ExampleTerminusJob:
    # This makes sure the session is cached before the parallel jobs start.
    needs: [TerminusGetProdSession]
    runs-on: ubuntu-22.04
    environment: Pantheon
    strategy:
      matrix: ['site-a', 'site-b', 'site-c']
    steps:
      - name: Log in to Terminus by restoring existing session or initiating new
        uses: lullabot/terminus-auth-with-session-cache@v2
        with:
          pantheon-machine-token: ${{ secrets.PANTHEON_MACHINE_TOKEN_PRODUCTION }}
          # Reuse existing session from cache, if possible.
          force-new-session: false
          # Set 'ddev: true' if using DDEV in GitHub Actions.
          ddev: false
      - name: Example step, clear caches on each site
        run: ddev terminus drush ${{ matrix.site-name }}.live -- cache:rebuild
        shell: bash
```

## Example workflow for refreshing tokens daily off-peak
If you have lots of workflows running simultaneously, you may opt for generating
new Terminus sessions during off-peak hours, so an old session doesn't expire at
a random moment when lots of jobs are running.
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
      - name: Refresh the DEV Terminus session in cache
        uses: lullabot/terminus-auth-with-session-cache@v2
        with:
          pantheon-machine-token: ${{ secrets.PANTHEON_MACHINE_TOKEN_DEV }}
          force-new-session: true
  TerminusGetProdSession:
    runs-on: ubuntu-22.04
    environment: Pantheon
    steps:
      - name: Refresh the PROD Terminus session in cache
        uses: lullabot/terminus-auth-with-session-cache@v2
        with:
          pantheon-machine-token: ${{ secrets.PANTHEON_MACHINE_TOKEN_PRODUCTION }}
          force-new-session: true
```

## DDEV
This action supports authenticating Terminus inside the
[DDEV](https://ddev.com/) container. Make sure DDEV is started before passing
"`ddev: true`" to this action.

Options for installing DDEV in GitHub Actions:

- https://github.com/marketplace/actions/setup-ddev-in-github-workflows
- https://github.com/Lullabot/drainpipe/blob/main/scaffold/github/actions/common/ddev/action.yml


## Pantheon documentation
The related Pantheon documentation lives here: [Authenticate Terminus in a
GitHub Actions Pipeline](https://docs.pantheon.io/terminus/ci/github-actions).
It currently recommends a different approach but hopefully will be updated soon
to reference this action or an equivalent one.
