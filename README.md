# Yarn CircleCI Monorepo
> A monorepo architecture built using Yarn Workspaces and CircleCI

This repository nods towards two other technology choices:

- AWS CloudFormation
- TypeScript

It works fine without them, but you may need to delete some of the boilerplate before getting started if you don't love them like we do.

## Yarn Workspaces
### Benefits
- Run `yarn` from anywhere to install dependencies across all services
- Each service can specify it's own dependencies, and version, which separates concerns and makes deployment bundles smaller
- Each service can use it's own build tools, such as TypeScript and WebPack

### Gotchas
#### Use `nohoist`
Inherently, Workspaces will try to hoist shared libraries to the root of the repository. To ensure all dependencies are bundles by our service's `yarn` scripts we use `nohoist` to leave them all in the subdirectories.

This seemed like a fine tradeoff because we specifically want to deploy services separately, so de-duplication of dependencies across services wouldn't benefit us anyway.

#### Sharing Files Isn't Easy
For shared logic, it's gotta be split out into it's own workspace package. This is again due to needing all source files inside the service directory for bundling. A workaround could be to Webpack our services and deploy just the bundles files, this would also give us tree-shaking.

This seemed like a fine tradeoff because shared logic probably should be it's own package anyway. Now we can version it and manage it's dependencies separate as well, which is not a bad practice.

## CircleCI
### Benefits
#### Build Flags
This architecture uses CircleCI's cache feature as a build flag to avoid rebuilding and redeploying services that haven't been updated. "Updated", in this case, means "package.json" has changed, (new dependencies or bumped version). This is done for each service independently.

Note that code changes won't bust this cache, only `package.json` changes. Due to this, it is good practice to bump the version of the service whenever there is a code change. We use [semantic versioning](https://semver.org/). Here's how it works:

1. `restore-build-flag` is an *alias* that runs after checking out the repository. The only thing in this cache is a file called `build.flag`.
  ```yaml
  restore_cache:
    keys:
      - build-flag-{{ checksum "package.json" }}
  ```
2. `test-build-flag` is an *alias* that can be run right after the build flag is restored. If `build.flag` exists, CircleCI will skip the rest of this job.
  ```yaml
  run:
    name: Exit if build flag exists
    command: |
      FILE=build.flag
      if test -f "$FILE"; then
          echo "$FILE exist"
          circleci step halt
      fi
  ```
3. `save-build-flag` is a *command* that is run *at the end of a successful build* to prevent the service from being rebuilt/deployed.
  ```yaml
  save-build-flag:
    steps:
      - run:
          name: Create build flag
          command: touch build.flag
      - save_cache:
          paths:
            - build.flag
          key:
            build-flag-{{ checksum "package.json" }
  ```

### Cache
In addition to using the cache as a build and deploy flag, we can also use cache for cache! This process is run for each service independently.

CircleCI cache often captures the `/node_modules/` directory and is built from the `yarn.lock` file, which we don't have in each service directory thanks to Yarn Workspaces. Fortunately, we can generate them. Here's how it works:

1. `generate-lock-file` is an *alias* which runs after the build flag has been tested.
  ```yaml
  run:
    name: Generate lock file
    command: yarn generate-lock-entry >> yarn.lock
  ```

2. `restore-cache` is an *alias* which will pull in `/node_modules/` based on the `yarn.lock`. This saved a lot of time in builds during the `yarn install` step.
  ```yaml
    restore_cache:
      keys:
        - dependencies-cache-{{ checksum "yarn.lock" }}
  ```

3. `save-cache` is an *alias* that should be run after `yarn install` has done the heavy lifting for us.
  ```yaml
    save_cache:
      paths:
        - node_modules
      key: dependencies-cache-{{ checksum "yarn.lock" }}
  ```

### Gotchas
Our CircleCI usage doesn't have as many quirks as Yarn Workspaces, but a few things come to mind:

1. We're grossly misusing cache as a build and deploy flag
2. Our job control is limited so more complex workflows may not be able to leverage the `test-build-flag` alias
3. A service's `package.json` must change to bust the build flag cache; generally speaking, that means bumping the version of the service at the least
3. CircleCI's API is limited
    - We can't tell Circle that we skipped the build, it assumes the step succeeded which is a little misleading
    - We can't easily retry a step that we "skipped" due to a build flag, we have to go bump the service's version
