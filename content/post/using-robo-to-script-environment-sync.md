---
title: "Using Robo to Script Environment Sync"
date: 2017-10-12T19:04:20-04:00
draft: false
---

In my day to day work developing for Drupal my colleagues and I noticed we would often have to synchronize our local development sites with the currently deployed code/database/files in production/staging. It is critical to test your code against an environment that matches your canonical server as closely as possible. There are a couple of reasons for this:

1. When your code is applied in production during deploy, it needs to work flawlessly. This means that before you wrap up a feature, it should be tested against the current state of the production site. For a typical Drupal 7 project, if you just made some changes to the site, you likely changed the database and exported your config to features. Your local database is now ahead of production. After exporting your config to code, you need to reset your database and files and revert features or apply database updates. You will frequently discover in this process that you forgot a feature setting and that you need to re-export. Its a good thing you pulled production and checked before you deployed isn't it?

2. During development, you may be switching features rapidly and picking up different issues to handle. Between each feature change you need to make sure you start fresh, with an identical environment to production. If you end up chaining feature branches together, they may have to be tested and eventually deployed in exact sequence. If you instead base all of your new features off the master branch and canonical database, they should be able to be considered and deployed separately.

### Common Solution
So typically the enterprising developer who doesn't want to spend the time doing all of this by hand will write herself a handy bash script and go refill her coffee while the site syncs down. It saves having to remember the full aliases, drush option flags, and extra commands that you typically don't think about like enabling dev modules or setting some variables in the database. Her script probably looks something like this:

```sh
#! /bin/bash

drush sql-drop -y
drush sql-sync @app.dev @self -y
drush en devel views_ui context_ui flaming_unicorns stage_file_proxy -y
drush dis acquia_spi memcache redis -y
drush vset stage_file_proxy_origin http://example.com
```

This certainly gets the job done. That is pretty much what you would have typed for a typical sync operation.

### So...ain't broke...why fix it?
There is nothing really wrong with the bash script above (except perhaps my own personal choice of drush executions) but there are two potential issues depending on who you are as a developer:

You are not a shell programmer, you are more than likely a PHP developer. The above shell script is written in bash. For basic stuff like this. you likely have no issues. Those commands are exactly what you would type in order. What happens though if you want to abstract your script? What happens if you want conditionals and user prompts? Do you know how to do all that in BASH? Most developers that I speak with would much rather write in their preferred scripting language than spent 3 times as long figuring out Bash's if statement syntax and evaluating conditionals, variable replacement.

This script file is one big freight train of action. It lacks the robust and compartmentalized approach of the OO classes that make modern PHP, Ruby, etc so much fun and so flexible. If you are the type who embraces the changes coming in Drupal 8 and the dawn of the new age of the Elephant; you know what a difference it is writing drab procedural code vs. writing a really well conceived set of objects and expertly plugging them into an app.

### Gad Zooks! What do we do?
Some time ago I came across Robo: a robust commandline task runner for PHP. I really wanted something native to PHP to handle basic tasks and scripting and Robo fits the bill perfectly for me. Robo is written by the guy who maintains Codeception and I have really enjoyed it for several months now. I also pull in the robo-drush package for drupal sites because it makes things a bit simpler. Below is a generalized version of the scripts I use to sync my local site with remote servers. This script is admittedly far bigger than the short little bash script above, but it is also a lot more flexible; and it is just PHP. Take a look at the Robo docs for more ideas, I hope you find it useful!

```php
<?php
class RoboFile extends \Robo\Tasks {
  use Boedah\Robo\Task\Drush\loadTasks;

  /**
   * Path to Drush executable.
   */
  protected $drushBin;

  /**
   * Path to Drupal root.
   */
  protected $drupalRoot;

  /**
   * RoboFile constructor.
   */
  public function __construct() {
    $this->disable_prod = [
      'foo_module',
    ];

    $this->enable_dev = [
      'context_ui',
      'views_ui',
      'field_ui',
      'devel',
      'diff',
      'stage_file_proxy'
    ];

    $this->envs = [
      'dev' => 'https://dev.example.com/',
      'test' => 'https://test.example.com/',
      'prod' => 'https://example.com'
    ];

    $this->drushBin = (__DIR__) . '/vendor/bin/drush';
    $this->drupalRoot = (__DIR__) . '/docroot';
  }

  /**
   * Synchronizes local development environment with a specified remote.
   * @param $env
   * @throws \Robo\Exception\TaskException
   */
  public function sync($env) {
    if (!array_key_exists($env, $this->envs)) {
      return $this->say("I'm sorry but that environment is invalid");
    }
    if ($this->syncDb($env)->run()->wasSuccessful()) {
      $this->disableProdModules();
      $this->enableDevModules();
      $this->buildDrushTask()
        ->exec("vset stage_file_proxy_origin {$this->getEnvUrl($env)}")
        ->run();
    }

    if ($this->ask("Should I run updates and revert features? (y/n) \n") === 'y') {
      $this->runUpdates();
    }
  }

  /**
   * Run database updates and revert features.
   */
  public function runUpdates() {
    $this->buildDrushTask()
      ->maintenanceOn()
      ->updateDb()
      ->revertAllFeatures()
      ->maintenanceOff()
      ->run();
  }

  /**
   * @param $env
   * @return $this
   */
  private function syncDb($env) {
    return $this->buildDrushTask()
      ->exec("sql-drop")
      ->exec("sql-sync @app.{$env} @self");
  }

  /**
   * Builds a Drush task with common arguments.
   *
   * @return $this
   *    A DrushStack task to use in building a task.
   */
  private function buildDrushTask() {
    return $this->taskDrushStack($this->drushBin)
      ->drupalRootDirectory($this->drupalRoot);
  }

  /**
   * @return mixed
   */
  public function disableProdModules() {
    return $this->buildDrushTask()
      ->exec("dis {$this->getDisableProd()}")
      ->run();
  }

  /**
   * Get a list of modules to disable to pass to drush
   * @return string
   */
  private function getDisableProd() {
    return implode(' ', $this->disable_prod);
  }

  /**
   * @return mixed
   */
  public function enableDevModules() {
    return $this->buildDrushTask()
      ->exec("en {$this->getEnableDev()}")
      ->run();
  }

  /**
   * Get a list of modules to enable to pass to drush
   * @return mixed
   */
  private function getEnableDev() {
    return implode(' ', $this->enable_dev);
  }

  /**
   * @param $env
   * @return mixed
   */
  private function getEnvUrl($env) {
    return $this->envs[$env];
  }
}
```
