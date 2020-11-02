---
title: EAS builds on the CI
---

This document outlines how to trigger builds on EAS for your app from a CI environment.

## Configuring the app

To be able to trigger builds on EAS from the CI environment, first, we need to ensure that our app is configured to build on EAS and that we can trigger builds from our local machine.
To automatically configure your native project for building with EAS Build on Android and iOS, you will need to run the following command in the root of your project:

```sh
expo eas:build:init
```

See [EAS Build from scratch in 5 minutes](../eas-build-in-5-minutes/) for detailed explanation and examples of what this does.

## Triggering builds

To start triggering EAS builds on the CI, we need to install and configure `expo-cli` and then trigger new builds with the `eas:build` command.

### Prepare Expo CLI

To interact with the Expo API, we need to install the Expo CLI.
You can use an environment with this library preinstalled, or you can add it to the project as a development dependency.
The latter is the easiest way but might increase the installation time.
For vendors that charge you per minute, it might we worth creating a prebuilt environment.

To install the Expo CLI into your project, you can execute this script.

```sh
npm install --save-dev expo-cli
```

### Prepare authentication

Next, we will configure the publishing step of your application to Expo.
Before we can do this, we need to authenticate as the owner of the app.
This is possible by storing the username or email with the password in the environment.
Every vendor has implemented their storage mechanism for sensitive data like passwords, although they are very similar.
Most vendors make use of environment variables which are "injected" into the environment and scrubbed from the logs to keep it safe.

To perform the authentication, we will add this script to our configuration:

```sh
npx expo login -u <EXPO USERNAME> -p <EXPO PASSWORD>
```

If you don't want to expose the password in the login script, set the `EXPO_CLI_PASSWORD` environment variable to the password and run the following script instead:

```sh
npx expo login --non-interactive -u <EXPO USERNAME>
```

### Trigger new builds

After having the CLI library and authentication in place, we can finally create the build step.

To trigger new builds, we will add this script to our configuration:

```sh
npx expo eas:build --platform all --non-interactive
```

This will trigger a new build on EAS and print the URLs for the built files after the build completes.

<details><summary>Travis CI</summary>
<p>

```yaml
---
language: node_js
node_js:
  - node
  - lts/*
cache:
  directories:
    - ~/.npm
before_script:
  - npm install -g npm@latest

jobs:
  include:
    - stage: deploy
      node_js: lts/*
      script:
        - npm ci
        - npx expo login -u $EXPO_USERNAME -p $EXPO_PASSWORD
        - npx expo eas:build --platform all --non-interactive
```

> Put this into `.travis.yml` in the root of your repository.

</p>
</details>

<details><summary>Gitlab CI</summary>
<p>

```yaml
image: node:alpine

cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - ~/.npm

stages:
  - build

before_script:
  - npm ci

eas-build:
  stage: build
  script:
    - apk add --no-cache bash
    - npx expo login -u $EXPO_USERNAME -p $EXPO_PASSWORD
    - npx expo eas:build --platform all --non-interactive
```

> Put this into `.gitlab-ci.yml` in the root of your repository.

</p>
</details>

<details><summary>Bitbucket Pipelines</summary>
<p>

```yaml
image: node:alpine

definitions:
  caches:
    npm: ~/.npm

pipelines:
  default:
    - step:
        name: Build app
        deployment: test
        caches:
          - npm
        script:
          - apk add --no-cache bash
          - npm ci
          - npx expo login -u $EXPO_USERNAME -p $EXPO_PASSWORD
          - npx expo eas:build --platform all --non-interactive
```

> Put this into `bitbucket-pipelines.yml` in the root of your repository.

</p>
</details>

<details><summary>CircleCI</summary>
<p>

```yaml
version: 2.1

executors:
  default:
    docker:
      - image: circleci/node:10
    working_directory: ~/my-app

commands:
  attach_project:
    steps:
      - attach_workspace:
          at: ~/my-app

jobs:
  eas_build:
    executor: default
    steps:
      - checkout
      - attach_project

      - run:
          name: Install dependencies
          command: npm ci

      - run:
          name: Login into Expo
          command: npx expo-cli login -u $EXPO_USERNAME -p $EXPO_PASSWORD

      - run:
          name: Trigger build
          command: npx expo-cli eas:build --platform all --non-interactive

workflows:
  build_app:
    jobs:
      - eas_build:
          filters:
            branches:
              only: master
```

> Put this into `.circleci/config.yml` in the root of your repository.

</p>
</details>

<details><summary>GitHub Actions</summary>
<p>

```yaml
name: EAS Build
on:
  push:
    branches:
      - master
  workflow_dispatch:

jobs:
  build:
    name: Install and build
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v2

      - name: Setup Node.js
        uses: actions/setup-node@v1
        with:
          node-version: 10.x

      - name: Setup Expo
        uses: expo/expo-github-action@v5
        with:
          expo-version: 3.x
          expo-username: ${{ secrets.EXPO_CLI_USERNAME }}
          expo-password: ${{ secrets.EXPO_CLI_PASSWORD }}
          expo-cache: true

      - name: Install dependencies
        run: npm ci

      - name: Build on EAS
        run: expo eas:build --platform all --non-interactive
```

> Put this into `.github/workflows/eas-build.yml` in the root of your repository.

</p>
</details>
