---
id: 8d09cc3f-25e3-42a2-ac86-82806c0c8c65
title: Startup
desc: ""
updated: 1643349229406
created: 1610160007286
---

## Summary

- check if we are in a dendron workspace
  - workspace can be a code workspace (a .code-workspace file) or a native workspace (folder with dendron.yml)
- depending on what sort of workspace we're in, Dendron will instantiate either a Dendron[Native|Code]Workspace
  - if multiple folders have `dendron.yml`, dendron will resolve to the first one found. logic [here](https://github.com/dendronhq/dendron/blob/6035eed562eb3eb38de50722b1185927eb54a7c8/packages/plugin-core/src/_extension.ts#L264-L264)
- start engine server (local server that does indexing)
  - the engine server is spawned in a separate process
  - NOTE: if in DEV mode (when launching plugin using vscode build task), we do not spawn a separate process
- reload workspace (connect to the local server)
- activate file watchers

## Steps

### Create Dendron Workspace

An instance of DendronWorkspace is created near the beginning of activation. This is a singleton god object that exists as long as Dendron is running. The only requirement for initialization is the location of Dendron's workspace root. Example initilization below

```ts
new DendronCodeWorkspace({
  wsRoot: path.dirname(DendronExtension.workspaceFile().fsPath),
  logUri: context.logUri,
  assetUri,
});
```

Keep in mind that with [[Native Workspaces|rfc.31-native-workspace]],
the root of the VSCode workspace may not be the root of the Dendron workspace.
If the `dendron.yml` file is located in a subdirectory of the root, then that
directory is the root for Dendron. Use `findWSRootInWorkspaceFolders` to find
the actual root.

### Check for workspace

- [[../packages/plugin-core/src/_extension.ts]]

```ts
_activate {
    ws := DendronExtension.getOrCreate

    if ws.isActive {

        setupSegmentClient
        // see [[Run Migration|dendron://dendron.docs/pkg.dendron-engine.t.upgrade.arch.lifecycle#run-migration]]
        changes = runMigrationsIfNecessary

        if !changes {
            runConfigMigrationIfNecessary
        }
        // ---
        port = startServer
        updateEngineAPI(port)
        ws.reloadWorkspace // 312
        ws.activateWatchers
        ...
        // register all commands with vscode
        _setupCommands // 413
    }

}
```

### Reload Workspace

- src/commands/ReloadIndex.ts

```ts
// 49
reloadWorkspace {
    ws.reloadWorkspace
    if isFirstInstall {
        showTutorial
    }
    postReloadWorkspace
    emit(extension, initialized)
}
```

#### postReloadWorkspace

```ts
postReloadWorkspace {
    previousWsVersion = config.get(WORKSPACE_STATE.WS_VERSION)

    if previousWsVersion == 0.0.0 {
        execute(Upgrade)
        config.set(WORKSPACE_STATE.WS_VERSION, ws.version)
    } else {
        newVersion :=
        if (newVersion > previousWsVersion) {
            execute(Upgrade)
            config.set(WORKSPACE_STATE.WS_VERSION, ws.version)
            emit(extension, upgraded)
        }
    }

}

```

## showWelcome

```ts
showWelcomeOrWhatsNew {
    switch(extensionInstallStatus) {
        case INITIAL_INSTALL {
            showWelcome
        }
        ....
    }
}

showWelcome {
    welcomeContent :=
    view = createWebviewPanel(welcomeContent)

    view.on(msg => {

        switch(msg {
            case initializeWorkspace {
                SetupWorkspaceCommand().run(workspaceInitializer: TutorialInitializer)
            }
        })

    })
}
```

## SetupWorkspaceCommand

## Reference

### getOrCreate

```ts
if !_DendronWorkspace
    _DendronWorkspace = new DendronWorkspace
```

```ts
constructor {
    @_setupCommands
    ...
}
```

### startServer

- file: src/\_extension.ts:
- related: [[launch|pkg.dendron-api-server.internal#launch]]

```ts
startServer {
    maybePort :=
    if !maybePort {
        launch()
    }
}
```

### updateEngine

- [[workspace.vaults|pkg.plugin-core.internal#workspacevaults]]

- src/utils.ts

```
WSUtils.updateEngineAPI {
    EngineAPIService.create
}
```

```ts
class EngineAPIService {
    create {
        const vaults =
            DendronWorkspace.instance().vaults?.map((ent) => ent.fsPath) || [];
        DendronEngineClient.create({ vaults, ws, port, history })
    }
}
```

### workspace.vaults

```ts
get vaults {

}

```

### ws.reloadWorkspace

```ts
executeCommand(RELOAD_INDEX);
```
