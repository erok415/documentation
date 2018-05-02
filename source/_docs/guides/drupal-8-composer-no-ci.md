---
title: Drupal 8 and Composer on Pantheon Without Continuous Integration
description: Learn how to manage Drupal 8 using Composer with Pantheon.
tags: [moreguides]
categories: [drupal]
type: guide
permalink: docs/guides/:basename/
contributors:
  - andrewtaylor
  - dwayne
  - davidneedham
---

In an ideal situation, you might ignore the vendor directory and use [an automated build step with continuous integration](https://pantheon.io/docs/guides/build-tools/) to install your site with composer on the fly. But sometimes your project doesn’t have the budget or time needed to justify this workflow.

In this guide, we’re going to simplify things and run through the bare necessities to use [Composer](https://getcomposer.org/) to install your Drupal 8 site on your local machine and push that to Pantheon. We’ll then demonstrate how to add a module using the same process.

Using a Composer managed site **removes** the ability to [apply Drupal core updates via the site dashboard](https://pantheon.io/docs/core-updates/). This is for advanced users who are comfortable taking complete responsibility for the management of site updates with Composer.

**Note** this doc uses the `andrew-drops-8-composer` site slug. This will need to be replaced with your unique site slug.

## Creating the Pantheon Site

To begin, we’ll want to start a brand new Drupal 8 site on Pantheon from our empty upstream. This upstream is different from the Drupal 8 upstream in that it does not come with any Drupal files. As such, you must use Composer to download Drupal.

Create a new Pantheon site with an empty upstream

```
terminus site:create andrew-drops-8-composer 'Andrew Drops 8 Composer' empty
```

**Note** you can also add the `--org` argument to `terminus site:create` if you would like the site to be part of an organization. See `terminus site:create -h` for details and help.

## Cloning example-drops-8-composer locally

Instead of setting up `composer.json` manually it is easier to start with the [`example-drops-8-composer`](https://github.com/pantheon-systems/example-drops-8-composer) repository.

First, clone the `example-drops-8-composer` repository locally.

```
git clone git@github.com:pantheon-systems/example-drops-8-composer.git andrew-drops-8-composer
```

Change into the cloned directory.

```
cd andrew-drops-8-composer

ls
```

## Updating the git remote URL

Find the git URL for the Pantheon site created earlier.

```
terminus connection:info andrew-drops-8-composer.dev --field=git_url
```

Update the git remote to use the Pantheon site git URL returned rather than the `example-drops-8-composer` GitHub URL.

```
git remote set-url origin ssh://codeserver.dev.SITE_UUD@codeserver.dev.SITE_UUD.drush.in:2222/~/repository.git
```

## Removing Automation Pieces
`example-drops-8-composer` was designed to run automated tests on a continuous integration server. Since contonuous integration won't be used you can safely remove the directories below.

* `script/github`
* `script/gitlab`
* `.circleci`

If you don't plan on running automated tests locally you can completely remove the testing functionality.

<div class="panel panel-drop panel-guide" id="accordion">
  <div class="panel-heading panel-drop-heading">
    <a class="accordion-toggle panel-drop-title collapsed" data-toggle="collapse" data-parent="#accordion" data-proofer-ignore data-target="#unique-anchor">
      <h3 class="info panel-title panel-drop-title" style="cursor:pointer;"><span style="line-height:.9" class="glyphicons glyphicons-wrench"></span>Remove Test Suites</h3>
    </a>
  </div>
  <div id="unique-anchor" class="collapse" markdown="1" style="padding:10px;">
    First, delete the `tests` directory. Next, you will need to modify `composer.json`.

    * Remove all dependencies in the `require-dev` section
    * Update the `scripts` section to the following:

    ```
        "scripts": {
            "build-assets": [
                "@prepare-for-pantheon",
                "composer install --optimize-autoloader --no-dev"
            ],
            "drupal-scaffold": "DrupalComposer\\DrupalScaffold\\Plugin::scaffold",
            "prepare-for-pantheon": "DrupalProject\\composer\\ScriptHandler::prepareForPantheon",
            "post-install-cmd": [
                "@drupal-scaffold",
                "DrupalProject\\composer\\ScriptHandler::createRequiredFiles"
            ],
            "post-update-cmd": [
                "DrupalProject\\composer\\ScriptHandler::createRequiredFiles"
            ],
            "post-create-project-cmd": [
                "@drupal-scaffold",
                "DrupalProject\\composer\\ScriptHandler::createRequiredFiles"
            ]
        },
    ```
  </div>
</div>

## Managing Drupal with Composer

### Downloading Drupal Dependencies with Composer

Normally the next step would be going through the standard Drupal installation. But since we’re using Composer, none of the core files exist yet. Let’s use Composer to install Drupal core.

Since we modified `composer.json` we will need to run `composer update`. This may take a while as all of Drupal core and its dependencies will be downloaded. Subsequent updates should take less time.

![image of terminal running a composer install](/source/docs/assets/images/guides/drupal-8-composer-no-ci/drops-8-composer-install.png)

`git status`

The `example-drops-8-composer` `.gitignore` file assumes that you’re using a build step with continuous integration. To make it compatible with this method we need to remove everything above the CUT section. Without this modification critical components, such as Drupal core and contrib modules, will be ignored and not pushed to Pantheon. Now let’s run git status again to make sure everything is included.

`git status`

![image of git status showing the changed files in red](/source/docs/assets/images/guides/drupal-8-composer-no-ci/drops-8-composer-git-status-after-installing-d8.png)

Set the site to git mode.

`terminus connection:set andrew-drops-8-composer.dev git`

Add and commit the code files. A git force push is necessary initially but subsequent pushes should not need `--force`.

```
git add .

git commit -m 'Drupal 8 install'

git push --force
```

**Note** the `vendor` directory is being committed to Pantheon. This is because Pantheon needs the full site artifact. If you prefer to ignore the `vendor` directory then take a look at our documentation on the more advanced automated workflow with a build step.

### Installing Drupal

Now that the code for Drupal core exists on our Pantheon site, we need to actually install Drupal.

Change to SFTP mode as files need to be writeable on Pantheon in order to install Drupal.

`terminus connection:set andrew-drops-8-composer.dev sftp`

Use Terminus Drush to install Drupal.

`terminus drush andrew-drops-8-composer.dev -- site-install -y`

Log in to your new Drupal 8 site to verify it is working.

`terminus drush andrew-drops-8-composer.dev -- uli`

Set the connection mode back to git

`terminus connection:set andrew-drops-8-composer.dev git`

### Adding a New Module with Composer

Next, let’s add a new module to our site. For this example, we’ll add the address module. We advocate working in feature branches on Pantheon, so let's create a git branch and spin up a Multidev environment.

```
git checkout -b addr-module

composer require "drupal/address ~1.0"

composer update

git add .

git commit -m "Adding the address module with Composer"

git push -u origin addr-module
```

Spin up a Multidev environment from the git branch we just pushed up to Pantheon.

`terminus multidev:create andrew-drops-8-composer.dev addr-module`

![image of multidev creation](/source/docs/assets/images/guides/drupal-8-composer-no-ci/drops-8-composer-multidev-creation.png)

Log in to your new environment and verify that the address module exists.

`terminus drush andrew-drops-8-composer.addr-module -- uli`

![image of installing address module](/source/docs/assets/images/guides/drupal-8-composer-no-ci/drops-8-composer-drupal-8-address-module-install.png)

### Managing Drupal Updates with Composer
Just like adding a new module updates to existing Composer managed third-party items (Drupal core, contrib modules and themes) will need to be done locally.

You can run `composer update` to download all available updates within the constraints defined in `composer.json`. You can update specific dependencies only by listing them explicitly in the `composer update` commands. 

For example, to update Drupal core you would use `composer update drupal/core`. If `composer.json` had the version constraint for `drupal/core` at `^8` then Composer will update Drupal core to the latest version of `8` but not update to `9.x`. You can read more about version constraints in the [version constraints documentation](https://getcomposer.org/doc/articles/versions.md#caret-version-range-).

Once the desired dependencies have been updated with Composer you will need to commit the new files to Pantheon.

#### Congratulations! You now have a Drupal 8 site on Pantheon that is managed by Composer.