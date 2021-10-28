# deployer-action by qlicks



## Deployer action for build deploy and create stage for magento 2.x

### Features:

- Build project 

- Integrated custom enviroment variables

- Integrated composer cache

- Deploy production or stage

- Notifaction to slack with step status

- Auto backups databases

  
  
  

  #### Build project and generate composer diff example github action configuration:

  ```yaml
  name: CI
  
  on:
    pull_request:
        branches: [master, STAGE]
        
    workflow_dispatch:
  
  jobs:
    composer-diff:
      name: Composer Diff
      runs-on: self-hosted
      steps:
        - id: Checkout
          uses: actions/checkout@v2
          with:
            fetch-depth: 0
        - name: Deployer Get composer.lock from server
          uses: qlicks/deployer-action@master
          env: 
              SSH_PRIVATE_KEY: ${{ secrets.SSH_KEY }}
          with:
              task: composer:lock:get 
              target: master
              extra_arguments: -vvvv
                    
        - name: Build
          uses: qlicks/deployer-action@master
          env: 
              COMPOSER_AUTH: ${{ secrets.AUTH }}
          with:
              task: build
              target: build
              extra_arguments: -vvvv
              
        - name: Generate composer diff
          id: composer_diff 
          uses: qlicks/composer-diff-action@master
          env: 
              GITHUB_TOKEN: ${{ secrets.TOKEN }}
          with:
              base: "./old_composer.lock"
              target: "./composer.lock"
              with-links: true
              
        - uses: marocchino/sticky-pull-request-comment@v2
          with:
            header: composer-diff
            path: composer.diff
  ```

  #### Build and deploy magento to server example configuration:

```yaml
name: CD

on:
  workflow_dispatch:
    inputs:
      stage:
         description: 'Deploy to production or STAGE'
         required: true
         default: 'STAGE'

env:
  COMPOSER_AUTH: ${{ secrets.AUTH }}
  SLACK_WEBHOOK_URL:  ${{ secrets.SLACK_WEBHOOK }}
  SSH_PRIVATE_KEY: ${{ secrets.SSH_KEY }}

jobs:
  build:
    runs-on: self-hosted
    steps:
      - id: Checkout
        uses: actions/checkout@v2
        with:
          fetch-depth: 0
      - name: Build
        uses: qlicks/deployer-action@v1
        with:
            task: build
            target: build
            slack_channel: deploy-alerts

      - uses: actions/upload-artifact@v2
        with:
            name: artifacts
            path: artifacts/**
      - uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          channel: '#deploy-alerts'
          fields: repo,message,commit,author,action,eventName,ref 
        if: failure() 
            
  deploy:
    runs-on: self-hosted
    needs: build
    steps:
      - id: Checkout
        uses: actions/checkout@v2
        with:
          fetch-depth: 0
      - uses: actions/download-artifact@v2
        with:
            name: artifacts
            path: artifacts/
            
      - name: Deploy
        uses: qlicks/deployer-action@v1
        with:
            task: deploy-artifact
            target: ${{ github.event.inputs.stage }}
            slack_channel: deploy-alerts
            
      - uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          channel: '#deploy-alerts'
          fields: repo,message,commit,author,action,eventName,ref 
        if: always() 
  ```

  

  ### For host configuration you must put file hosts.yml to root folder of project

  ##### Example hosts.yml

  ```yaml
  import: recipe/common.php
  
  config:
    bin/composer: /usr/local/bin/composer2 
    bin/php: /usr/local/bin/php74 
    artifact_path: artifacts
    artifact_file: artifact.tar.gz
    magento_dir: .
    dos2unix: true
    keep_releases: 5
    keep_db_backups: 1
    static_deploy_options: --jobs=4 -f
    languages: en_US nl_NL
    cache_enabled_caches: all
    default_timeout: 1200
    artifact_excludes_file:
      - deploy.php
      - artifacts
    shared_files:
      - {{magento_dir}}/app/etc/env.php
      - {{magento_dir}}/app/etc/config.php
    shared_dirs:
      - {{magento_dir}}/pub/media
      - {{magento_dir}}/var/log
      - {{magento_dir}}/var/report
      - {{magento_dir}}/var/export
      - {{magento_dir}}/var/session
  
  
  hosts:
    production:
      labels: 
         stage: 
           - master
           - production
        hostname: example.com
        remote_user: username
        deploy_path: /var/www/
        port: 339
        ssh_multiplexing: false
        ssh_arguments: 
           - '-o UserKnownHostsFile=/dev/null'
           - '-o StrictHostKeyChecking=no'
    
    stage:
        labels: 
           stage: STAGE
        hostname: example.com
        remote_user: username
        deploy_path: /var/www/
        port: 339
        ssh_multiplexing: false
        ssh_arguments: 
           - '-o UserKnownHostsFile=/dev/null'
           - '-o StrictHostKeyChecking=no'
    tasks:
      deploy:
        - deploy:prepare
        - deploy:vendors
        - deploy:publish
  
      deploy:vendors:
        script: 'cd {{release_path}} && echo {{bin/composer}} {{composer_options}} 2>&1'
  ```
```bin/composer``` - Path to composer binary used when build project. 

*Default:* /usr/local/bin/composer (Composer version 1)

Optional: /usr/local/bin/composer2 (Composer version 2)

```bin/php``` - Path to php interpretator on server 

*Default:* System default path 

Optional: On hipex need set to ~/./bin/php

```dos2unix``` - enable convert all files for dos to unix on build step 

```keep_releases``` - How mush releases keep in enviroment

*Default:* 5

```keep_db_backups``` - How many backups store on server

*Default:* 1

0 - Disable feature

```static_deploy_options``` - Arguments for magento static build

*Default:* --jobs=4 -f

```labels``` - Enviroment labels can be any word

