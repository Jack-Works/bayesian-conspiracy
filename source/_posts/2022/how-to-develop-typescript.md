---
title: 'How to develop TypeScript'
toc: true
categories: frontend
date: 2022-01-02 10:13:12
update: 2022-11-22 19:51:12
lang: en
tags:
    - typescript
---

This article contains my most used workflow when working on the TypeScript repo with VSCode.

<!-- more -->

> Update on Nov 2022: I have updated this article to reflect [the latest changes in the TypeScript repo](https://github.com/microsoft/TypeScript/pull/51387).

# Normal development workflow

## Build commands

- Core compiler: `npx hereby watch-tsc` / `npx hereby tsc`
- Language service: `npx hereby watch-tsserver` / `npx hereby tsserver`
- ...or both: `npx hereby watch-min` / `npx hereby min`

## Tests

- Run all tests: `npx hereby runtests-parallel`
- Run tests that name matches "class": `npx hereby runtests -t=class`
- Run test failed in the last run: `npx hereby runtests --failed`
- Accept new test results: `npx hereby baseline-accept`
- Run linter: `npx hereby lint`

## Add a new error message

- Open `src/compiler/diagnosticMessages.json`.
- Add the message you want.
- Run `npx hereby generate-diagnostics`.
- If VSCode doesn't show the new error message in the `Diagnostics` object, open `src/compiler/diagnosticInformationMap.generated.ts` in VSCode to load the latest result.

# Debug

## Debug `tsc`

1. Open a debug terminal.
![A menu that "Open JavaScript Debug Terminal" is highlighted.](debug-terminal.png)

1. Run `tsc` by `node ./built/local/tsc.js`.
![Debugging command line compiler](debug-tsc.png)

## Debug language server

### First-time setup

1. Create a folder at `../vscode-debug`.

1. Add a `.vscode/launch.json` with [the content in the gist](https://gist.github.com/Jack-Works/80d69633a55af03cec1f1d6328ffa9d1#file-vscode-launch-json).

1. Switch to the "Debug and Run" panel, start the debug profile. It should open a new VSCode window. ![A screenshot of the "Debug and Run" panel, with an arrow pointing to the "start debug" icon.](launch-vscode.png)

1. Open the JSON settings of the newly opened VSCode, [add some recommended settings](https://gist.github.com/Jack-Works/80d69633a55af03cec1f1d6328ffa9d1#file-vscode-debug-profile-settings-json).

1. Add a `../vscode-debug/package.json` with [the content in the gist](https://gist.github.com/Jack-Works/80d69633a55af03cec1f1d6328ffa9d1#file-vscode-debug-package-json)

1. Create a TypeScript project in the `../vscode-debug`

### Debug routine

1. Launch the debug profile VSCode.
1. Select "Attach to VS Code TS Server via Port". !["Attach to VS Code TS Server via Port"](select-task.png)
1. It will open a prompt of which port you want to attach.
1. Type "local" to filter the result. ![Selecting port to attach](select-port.png)
1. Now you can debug using the breakpoint in the VSCode. ![Debugging ts server](debug-tsserver.png)

If you have an "unbound breakpoint", which means the editor doesn't handle the source map yet (it might never will). You should use the `debugger` statement to trigger it. ![An unbound breakpoint](unbound-breakpoint.png)

If you want to reload the debugging TS Server, run `> TypeScript: Restart TS server` command, then you may need to re-attach by the steps above.

![Restart tsserver](restart-ts-server.png)

## Debug tests

Follow the [debug language server](#Debug-language-server) set up, and you'll find "Mocha Tests (currently opened test)" in the debug targets.

Open a test you want to debug (e.g. `tests/cases/compiler/2dArrays.ts`), and start the debug.

![Debugging tests](debug-test.png)
