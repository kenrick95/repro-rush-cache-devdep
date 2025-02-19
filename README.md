# Repro

## Situation

There are three packages:

- `pkg-a`:
    - scripts: `_phase:meow`: `pkg-c/bin.js`
    - dependencies: `pkg-b`
    - devDependencies: `pkg-c`
- `pkg-b`:
    - scripts: `_phase:meow`: `pkg-c/bin.js`
    - dependencies: None
    - devDependencies: `pkg-c`
- `pkg-c`:
    - dependencies: None
    - devDependencies: None

`build-cache.json`: `buildCacheEnabled` is set to true

`command-line.json` provides ` command:

```json
  "commands": [
    {
      "commandKind": "phased",
      "name": "meow",
      "summary": "Meow",
      "phases": ["_phase:meow"],
      "enableParallelism": true,
      "incremental": true
    }
  ],
  "phases": [
    {
      "name": "_phase:meow",
      "dependencies": {},
      "allowWarningsOnSuccess": true,
      "missingScriptBehavior": "silent"
    }
  ]
```

`rush-project.json` is set for all packages to be the same:

```json
  "operationSettings": [ 
    {
      "operationName": "_phase:meow",
      "outputFolderNames": []
    }
  ]
```

## Steps to Reproduce

1. `rush install`
2. `rush meow`

Observe the output

    ```log

    Rush Multi-Project Build Tool 5.148.0 - https://rushjs.io
    Node.js version is 20.18.1 (LTS)


    Starting "rush meow"

    Analyzing repo state... DONE (0.03 seconds)

    Executing a maximum of 10 simultaneous processes...

    ==[ pkg-b (meow) ]=================================================[ 1 of 2 ]==
    "pkg-b (meow)" completed successfully in 0.41 seconds.

    ==[ pkg-a (meow) ]=================================================[ 2 of 2 ]==
    "pkg-a (meow)" completed successfully in 0.56 seconds.



    ==[ SUCCESS: 2 operations ]====================================================

    These operations completed successfully:
    pkg-a (meow)    0.56 seconds
    pkg-b (meow)    0.41 seconds

    ```

3. `rush meow`

    Observe the output that it's restored from cache

    ```log

    Rush Multi-Project Build Tool 5.148.0 - https://rushjs.io
    Node.js version is 20.18.1 (LTS)


    Starting "rush meow"

    Analyzing repo state... DONE (0.04 seconds)

    Executing a maximum of 10 simultaneous processes...

    ==[ pkg-b (meow) ]=================================================[ 1 of 2 ]==
    "pkg-b (meow)" was restored from the build cache.

    ==[ pkg-a (meow) ]=================================================[ 2 of 2 ]==
    "pkg-a (meow)" was restored from the build cache.



    ==[ FROM CACHE: 2 operations ]=================================================

    These operations were restored from the build cache:
    pkg-a (meow)    0.02 seconds
    pkg-b (meow)    0.02 seconds

    ```

4. Edit `pkg-c/bin.js` & save

    ```diff
    #!/usr/bin/env node
    - console.log('Meow');
    + console.log('Woof');
    ```

5. Run `rush meow`

    **They're still restored from cache**! and the output is still `Meow` instead of `Woof`

    ```log
    Rush Multi-Project Build Tool 5.148.0 - https://rushjs.io
    Node.js version is 20.18.1 (LTS)


    Starting "rush meow"

    Analyzing repo state... DONE (0.04 seconds)

    Executing a maximum of 10 simultaneous processes...

    ==[ pkg-b (meow) ]=================================================[ 1 of 2 ]==
    "pkg-b (meow)" was restored from the build cache.

    ==[ pkg-a (meow) ]=================================================[ 2 of 2 ]==
    "pkg-a (meow)" was restored from the build cache.



    ==[ FROM CACHE: 2 operations ]=================================================

    These operations were restored from the build cache:
    pkg-a (meow)    0.02 seconds
    pkg-b (meow)    0.02 seconds


    rush meow (0.09 seconds)
    ```

I understand that it's probably because of [this](https://rushjs.io/pages/maintainer/build_cache/):

> By default, the following inputs are incorporated into Rush's cache key. In other words, if any of these things changes, then the project must be rebuilt:
> - the hashes of source files under other workspace projects that are _dependencies_ of the project
(applies to cache restoration strategy but not output preservation strategy)

but shouldn't it depend on `devDependencies` too? How can I modify this strategy to include `devDependencies` too?
