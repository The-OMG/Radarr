# Radarr — native Google Drive read fork (`gdrive-native`)

Radarr counterpart of the native-Drive [Sonarr fork](https://github.com/The-OMG/Sonarr).
Serves the Drive-backed portion of the movie library directly via the Google Drive API +
a local sqlite index instead of an rclone FUSE mount. The full design (read-only decorator
over `IDiskProvider`, per-drive sqlite index, `parent_id` listing, metadata from Drive +
probe-once cache, union stays mounted for writes) is identical to the Sonarr fork — see its
`docs/FORK_GDRIVE.md`. This file only records the Radarr-specific deltas.

## Feature code (new files, never conflict)

- `src/NzbDrone.Core/Drive/*` — DriveModels, DriveConfig, DriveClient, DriveIndex,
  DriveIndexBuilder, DispatchDiskProvider, DriveStartupIndexer
- `src/NzbDrone.Host/DriveDispatchExtensions.cs` — DryIoc decorator registration
- One-line hooks: `.AddDriveDispatch()` in `Bootstrap.cs`; `Google.Apis.Drive.v3` in
  `Radarr.Core.csproj`

`DispatchDiskProvider` implements Radarr's `IDiskProvider`, which differs from Sonarr's by:
`FileInfo GetFileInfo(string)`, and `GetFileInfos`/`MoveFolder` take an extra `bool`.

## Base / toolchain

- Base pinned to upstream stable tag **`v6.2.1.10461`** (matches the running Whatbox build
  6.2.1). Track `v6.x` stable tags.
- **.NET 8** (not .NET 6 like Sonarr v4). The box's system SDK is 6.0, so build with the
  SDK at `/home/theomg/.dotnet8`:

```sh
export DOTNET_ROOT=/home/theomg/.dotnet8
/home/theomg/.dotnet8/dotnet build src/Radarr.sln -c Release -nodeReuse:false -maxcpucount:2
# runnable:
/home/theomg/.dotnet8/dotnet msbuild -restore src/Radarr.sln -p:Configuration=Release \
  -p:Platform=Posix -p:RuntimeIdentifiers=linux-x64 -t:PublishAllRids -nodeReuse:false -maxcpucount:2
# -> _output/net8.0/linux-x64/publish/Radarr  (launch with DOTNET_ROOT set)
```

## Config / run

`gdrive.json` in the data dir (SA file, `cloudRoot`, `localBranch`, `indexPath`, ranked
`drives`). Radarr root folders live under `~/cloud/Movies/Processed/*`, so a real deploy
needs a **movie** index (the same indexer over the movie team-drive(s), e.g.
`0ABEto_GQCb5vUk9PVA`). Build large indexes in `/dev/shm` — the box disk is IO-saturated and
on-disk 700k-row index builds stall (see the Sonarr doc / ops notes).

## Maintain

`scripts/sync-upstream.sh` rebases `gdrive-native` onto the latest stable tag with
`git rerere`, then builds. Conflict surface is the one Bootstrap line + the one csproj line;
everything else is new files. Remotes: `origin` = Radarr/Radarr (upstream RO),
`fork` = The-OMG/Radarr. Commit author `14554607+The-OMG@users.noreply.github.com`.
