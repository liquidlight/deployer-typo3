# Liquid Light Deployer

PHP Deployer package for deploying Liquid Light TYPO3 websites. It uses Gitlab CI to deploy to the host.

This is an extension package which uses [PHP Deployer](https://deployer.org/)

## Upgrading to Version 2

Version 2 brings with it Deployer 7, which requires some changes:

- In your `deploy.php`, change `hostname('client')` to `set('hostname', 'client')`
- **When pushing live for the first time after the upgrade** add a Gitlab CI variable of `DEPLOYER_FLAGS` with the value of `-o release_name=XXX -vvv` - where XXX is the current release number + 1. Once deployed, delete this variable. More details are in the [deployer docs](https://deployer.org/docs/7.x/UPGRADE#step-2-deploy)


## Options

### Hosts

Set the hostname of the server in the `deploy.php` file - the rest of the connection details should be in your `.ssh/config` file

```php
host('production')
  ->set('hostname', 'client.xxx')
;
```

### Set environment

It is advised you set an environment for extra config to be applied. This is set on the `host()` to allow for different envs for different hosts

```php
host('production')
  ->set('ll_deployer_environment', 'cpanel')
;
```

- `vps`
- `cpanel`

### Assets upload

Will upload `html/assets` if there as well as `app/*/Resources/Public`, but if you want others that are built then you need to specify them as an array

```php
set('ll_deployer_asset_paths', ['app/nlw/Resources/Public']);
```

## Setup

You need for this process:

- Site to be using composer
- Site to have a `package.json` file
- Site to not rely on any tech from Gizmo

1. Run a `composer update` and `npm update` (commit any changes to lock files)
2. Create a `deploy.php` in the root of the project - **see below** for example contents
   - Set the `deploy_path` to `/var/www/[domain]` (you may need to deploy to a parallel folder as an interim)
   - Make sure the default SSH user has read & write access to the `deploy_path` on the live server (`775`)
3. `composer req liquidlight/deployer`
4. Ensure the **npm `scripts` block** (below) is in your `package.json`
5. Add deployment stage to `.gitlab-ci.yml` (see below)
   - Update the environment URL
   - Verify [front-end asset build process](https://gitlab.lldev.co.uk/devops/gitlab-ci/-/blob/main/jobs/deployment/deployer.deploy.gitlab-ci.yml) is correct for the site
6. Ensure your `.env` (or `.env.local`) file has `INSTANCE="local"` in it (if using development server, this already exists)
7. Run `./vendor/bin/dep deploy:prepare production` - this makes the skeleton files on live and ensures you can connect
8.  On the live server - Populate the `shared` folder (located in your `deploy_path`) with folder & file structure of that below. Run the following in `shared`
    - `touch .env`
    - `mkdir -p var html/{fileadmin,typo3temp,uploads}/`
    - Add details to the `.env` file and make sure it has:
       - `INSTANCE="production"`
       - `TYPO3_DB_HOST="localhost"` (or wherever the database is hosted)
9.  On Gitlab - Add a CI/CD variable with the title/key of `DEPLOY_HOST_PRODUCTION` and the value being that of the SSH config valuer
    - Go to the repository
    - Click **Settings -> CI/CD** on the left
    - Expand **Variables** and click **Add Variable**
10.  Set a second CI variable of `DEPLOYER_FLAGS` to `-vvv` for maximum output
11.  On the live server, check that `git` and `composer` are installed
12. If there are any folders or files on the live server you want to keep (e.g. `blog` folder), these need to be moved into the `shared` folder and added to the `shared_files` array (see CST & Liquid Light as examples)
13. Commit all your changes and push to Gitlab (e.g. `Task: Add deployer for automated deployments`) and push to Gitlab
14. On Gitlab, click the ▶️ button and watch the logs for issues
    - You may need to set the `bin/php` or `bin/composer` paths (e.g. NLW)
    - The `http_user` may need to be set (e.g. CST)
    - The `writable_mode` might need to be changed
    - You may need to set `set('writable_use_sudo', false);` if there is no sudo
15. Once happy it is deploying correctly, remove the `DEPLOYER_FLAGS` CI variable
16. Update the file in `/etc/cron.d/[domain_name]` to point to the correct place (or update in cPanel)
17. Update the `apache` config (if converting an existing site) to point to `current/html` (for cPanel, repoint the `public_html` symlink to `www/current/html`)
18. Regenerate the [SSL certificate with Certbot](https://hub.lldocs.dev/sysadmin/debian/ssl-certificates?_highlight=cert#installing-and-generating-ssl-certificate) to point to the new web root

### Code Examples

#### `deploy.php`

```php
<?php

namespace Deployer;

require_once __DIR__ . '/vendor/sourcebroker/deployer-loader/autoload.php';
new \LiquidLight\Deployer\Loader(__DIR__);

/**
 * Hosts
 */

// Production
host('production')
    ->set('hostname', 'client.xxx')
	->set('ll_deployer_environment', 'cpanel')
	->set('deploy_path', '/var/www/[domain]')
;
```

### `package.json`

```
    "scripts": {
        "dev": "./node_modules/.bin/gulp watch --dev",
        "watch": "./node_modules/.bin/gulp watch",
        "build": "./node_modules/.bin/gulp compile",
        "gulp": "./node_modules/.bin/gulp"
    },
```

### `.gitlab-ci.yml`

```
###
# Gitlab CI
#
# @author Mike Street
# @date 06-2021
#
###

include:
  - project: 'devops/gitlab-ci'
    file: '.gitlab-ci-template.yml'

stages:
  - linting
  - deployment

# Define stages for jobs
php:coding-standards:
  stage: linting
  extends:
    - .php:coding-standards

js:eslint:
  stage: linting
  extends:
    - .js:eslint

scss:stylelint:
  stage: linting
  extends:
    - .scss:stylelint

production:deploy:
  stage: deployment
  environment:
    name: production
    url: [domain name - including https]
  extends:
    - .production:deploy
```
