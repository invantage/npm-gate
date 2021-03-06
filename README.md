# npm-gate [![Build Status](https://travis-ci.org/invantage/npm-gate.svg?branch=master)](https://travis-ci.org/invantage/npm-gate) [![npm version](https://badge.fury.io/js/npm-gate.svg)](https://badge.fury.io/js/npm-gate)

A hard fork of lint-staged to support complex pre-commit commands both per staged file and as a set

See the [original project here](https://github.com/okonet/lint-staged)

## Why

The original lint-staged project is focused on enforcing linting in pre-commit hook, this project aim at adding gating capability right inside npm and supported through git pre-commit hooks.
This script will run arbitrary npm and shell tasks with a list of staged files as an argument, filtered by a specified glob pattern.

## Installation and setup

1. `npm install --save-dev npm-gate husky`
1. Install and setup your linters just like you would do normally. Add appropriate `.eslintrc`, `.stylelintrc`, etc.
1. Update your `package.json` like this:
  ```json
  {
    "scripts": {
      "precommit": "npm-gate"
    },
    "npm-gate": {
      "*.js": ["eslint --fix", "git add"]
    }
  }
  ```

Now change a few files, `git add` some of them to your commit and try to `git commit` them.

See [examples](#examples) and [configuration](#configuration) below.

> I recommend using [husky](https://github.com/typicode/husky) to manage git hooks but you can use any other tool.

> **NOTE:**
>
> If you're using commitizen and having following npm-script `{ commit: git-cz }`, `precommit` hook will run twice before commitizen cli and after the commit. [This buggy behaviour is introduced by husky](https://github.com/okonet/lint-staged/issues/152#issuecomment-306046520).
>
> To mitigate this rename your `commit` npm script to something non git hook namespace like, for example `{ cz: git-cz }`

## Configuration

Starting with v3.1 you can now use different ways of configuring it:

* `npm-gate` object in your `package.json`
* `.npmgaterc` file in JSON or YML format
* `npm-gate.config.js` file in JS format

See [cosmiconfig](https://github.com/davidtheclark/cosmiconfig) for more details on what formats are supported.

npm-gate supports simple and advanced config formats.

### Simple config format

Should be an object where each value is a command to run and its key is a glob pattern to use for this command. This package uses [minimatch](https://github.com/isaacs/minimatch) for glob patterns.

#### `package.json` example:
```json
{
  "scripts": {
    "my-task": "your-command"
  },
  "npm-gate": {
    "*": "my-task"
  }
}
```

#### `.npmgaterc` example

```json
{
  "*": "my-task"
}
```

This config will execute `npm run my-task` with the list of currently staged files passed as arguments.

So, considering you did `git add file1.ext file2.ext`, npm-gate will run the following command:

`npm run my-task -- file1.ext file2.ext`

### Advanced config format
To set options and keep npm-gate extensible, advanced format can be used. This should hold linters object in `linters` property.

## Options

* `linters` — `Object` — keys (`String`) are glob patterns, values (`Array<String> | String`) are commands to execute.
* `gitDir` — Sets the relative path to the `.git` root. Useful when your `package.json` is located in a subdirectory. See [working from a subdirectory](#working-from-a-subdirectory)
* `concurrent` — *true* — runs linters for each glob pattern simultaneously. If you don’t want this, you can set `concurrent: false`
* `chunkSize` — Max allowed chunk size based on number of files for glob pattern. This is important on windows based systems to avoid command length limitations. See [#147](https://github.com/okonet/lint-staged/issues/147)
* `subTaskConcurrency` — `2` — Controls concurrency for processing chunks generated for each linter.
* `verbose` — *false* — runs npm-gate in verbose mode. When `true` it will use https://github.com/SamVerschueren/listr-verbose-renderer.
* `globOptions` — `{ matchBase: true, dot: true }` — [minimatch options](https://github.com/isaacs/minimatch#options) to customize how glob patterns match files.

## Filtering files

It is possible to run linters for certain paths only by using [minimatch](https://github.com/isaacs/minimatch) patterns. The paths used for filtering via minimatch are relative to the directory that contains the `.git` directory. The paths passed to the linters are absolute to avoid confusion in case they're executed with a different working directory, as would be the case when using the `gitDir` option.

```js
{
  // .js files anywhere in the project
  "*.js": "eslint",
  // .js files anywhere in the project
  "**/*.js": "eslint",
  // .js file in the src directory
  "src/*.js": "eslint",
  // .js file anywhere within and below the src directory
  "src/**/*.js": "eslint",
}
```

## Trapping the file name

It can be useful to avoid pushing the committed list of file at then end of the npm script/command.
Example: when using cucumberJS, you don't want the pre-commit to be treating all your committed javascript files as
gherkin feature files.
In that case, you can trap the files name list using a complex command, like this:
```js
{
  "*.js": "npm run test",
  "codeaway/**/*.js": { "command": "cucumber-js testcodeaway/features", "trap": true}
}
```
It will now launch your npm script/command if any \*.js files are committed in codeaway/ but without pushing the file list
in the command

## Pretty command name

You can also customize the displayed command name in the prompt, like this:
```js
{
  "*.js": {"name": "my very important command", "command": "npm run test" }
}
```


## Advanced commands (using a template)

For more advanced uses, a (light) templating system allows some flexibility regarding your command parsing.
All 'patterns' have to be enclosed in the '<' and '>'.
All enclosed expression will be repeated as many time as there is pre-commited files.
Inside the enclosure, 4 patterns are available to delimit the pre-commited filename expansion :
- <> or \<full\> : to insert the full filename as given by git
- \<filename\> : to insert only the filename (without the extension)
- \<path\> : to insert the path of the file (beware, git will use a relative path to the root of the repo)
- \<extension\> : to insert the extension

### Advanced command example
```js
{
  // compress binary files anywhere in a subdirectory /bin where the file is currently
  "*.bin": "compress <--in=<full> --out=<path>/bin/<filename>.tar.gz>",
  // optimize all SVG file using svgo and pushing each optimized version with 'opt' before the extension
  "*.svg": "svgo --multipass <-i <> -o <path><filename>.opt.svg>",
  // translation of the simple "eslint" command in an advanced command (those are equivalent)
  "*.js": "eslint -- <<>>"
}
```

## What commands are supported?

Supported are both local npm scripts (`npm run-script`), or any executables installed locally or globally via `npm` as well as any executable from your $PATH.

> Using globally installed scripts is discouraged, since npm-gate may not work for someone who doesn’t have it installed.

`npm-gate` is using [npm-which](https://github.com/timoxley/npm-which) to locate locally installed scripts, so you don't need to add `{ "eslint": "eslint" }` to the `scripts` section of your `package.json`. So  in your `.lintstagedrc` you can write:

```json
{
  "*.js": "eslint --fix"
}
```

Pass arguments to your commands separated by space as you would do in the shell. See [examples](#examples) below.

Starting from [v2.0.0](https://github.com/okonet/lint-staged/releases/tag/2.0.0) sequences of commands are supported. Pass an array of commands instead of a single one and they will run sequentially. This is useful for running autoformatting tools like `eslint --fix` or `stylefmt` but can be used for any arbitrary sequences.

## Reformatting the code

Tools like ESLint/TSLint or stylefmt can reformat your code according to an appropriate config  by running `eslint --fix`/`tslint --fix`. After the code is reformatted, we want it to be added to the same commit. This can be done using following config:

```json
{
  "*.js": ["eslint --fix", "git add"]
}
```


## Working from a subdirectory

If your `package.json` is located in a subdirectory of the git root directory, you can use `gitDir` relative path to point there in order to make lint-staged work.

```json
{
    "gitDir": "../",
    "linters":{
        "*": "my-task"
    }
}
```

## Examples

All examples assuming you’ve already set up npm-gate and husky in the `package.json`.

```json
{
  "name": "My project",
  "version": "0.1.0",
  "scripts": {
    "precommit": "npm-gate"
  },
  "npm-gate": {}
}
```

*Note we don’t pass a path as an argument for the runners. This is important since npm-gate will do this for you. Please don’t reuse your tasks with paths from package.json.*

### ESLint with default parameters for `*.js` and `*.jsx` running as a pre-commit hook

```json
{
  "*.{js,jsx}": "eslint"
}
```

### Automatically fix code style with `--fix` and add to commit

```json
{
  "*.js": ["eslint --fix", "git add"]
}
```

This will run `eslint --fix` and automatically add changes to the commit. Please note, that it doesn’t work well with committing hunks (`git add -p`).


### Automatically fix code style with `prettier` for javascript + flow or typescript

```json
{
  "*.{js,jsx}": ["prettier --parser flow --write", "git add"]
}
```

```json
{
  "*.{ts,tsx}": ["prettier --parser typescript --write", "git add"]
}
```


### Stylelint for CSS with defaults and for SCSS with SCSS syntax

```json
{
  "*.css": "stylelint",
  "*.scss": "stylelint --syntax=scss"
}
```

### Automatically fix SCSS style with `stylefmt` and add to commit

```json
{
  "*.scss": ["stylefmt", "stylelint --syntax scss", "git add"]
}
```

### Run PostCSS sorting, add files to commit and run Stylelint to check

```json
{
  "*.scss": [
    "postcss --config path/to/your/config --replace",
    "stylelint",
    "git add"
  ]
}
```
