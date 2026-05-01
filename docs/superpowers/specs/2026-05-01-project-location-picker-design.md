# Project Location Picker Design

## Goal

PowerShell should let a user persistently save project directories and quickly jump to one without repeatedly typing `Set-Location` or `cd`.

The daily entry point is `pj`:

```powershell
pj
```

With no argument, `pj` opens an interactive terminal picker that lists saved projects. The user can filter, move through the list, and press Enter to change the current PowerShell session to the selected project directory.

```powershell
pj PowerShell
```

With a project name, `pj` jumps directly to that project.

## Scope

The first version supports local file-system project directories only. It does not change PowerShell's default startup directory, and it does not automatically track every `Set-Location` call.

This keeps the feature explicit and avoids surprising scripts, tests, terminal integrations, remoting sessions, or users who rely on the existing current-directory behavior.

## User Experience

Users can manage saved locations with standard cmdlets:

```powershell
Add-ProjectLocation PowerShell
Add-ProjectLocation Blog E:\code\blog
Get-ProjectLocation
Remove-ProjectLocation Blog
```

`Add-ProjectLocation <Name>` saves the current file-system directory. `Add-ProjectLocation <Name> <Path>` saves the specified directory.

`Get-ProjectLocation` returns objects with at least:

```text
Name
Path
LastUsed
```

`Enter-ProjectLocation <Name>` changes the current session location to the saved path. `pj` is a built-in alias for `Enter-ProjectLocation`; with no name it launches the picker, and with a name it jumps directly.

## Commands

Initial cmdlets:

```powershell
Add-ProjectLocation [-Name] <string> [[-Path] <string>] [-Force]
Get-ProjectLocation [[-Name] <string>]
Enter-ProjectLocation [[-Name] <string>]
Remove-ProjectLocation [-Name] <string> [-Force]
```

`Add-ProjectLocation` validates that the path exists and is a file-system container. If a project with the same name exists, it fails unless `-Force` is supplied.

`Enter-ProjectLocation` validates that the saved path still exists before changing directory. If the path is missing, it reports an error and leaves the current location unchanged.

`Remove-ProjectLocation` removes a saved project by name.

## Picker Behavior

When `Enter-ProjectLocation` is called without `Name` in an interactive host, it opens a terminal picker.

The picker supports:

- Search/filter by project name and path.
- Keyboard navigation.
- Enter to select.
- Escape or Ctrl+C to cancel without changing location.

The picker should degrade cleanly in non-interactive hosts. If interaction is unavailable, `Enter-ProjectLocation` without `Name` writes a clear error instructing the user to pass a project name.

## Persistence

Project data is stored per user in `project-locations.json` under PowerShell's existing current-user configuration directory. On Windows this is the same directory family used for the current user's PowerShell configuration; on Unix-like systems it follows the PowerShell `XDG_CONFIG_HOME` behavior.

The JSON shape is:

```json
{
  "version": 1,
  "projects": [
    {
      "name": "PowerShell",
      "path": "E:\\code\\opencode\\PowerShell",
      "lastUsed": "2026-05-01T00:00:00Z"
    }
  ]
}
```

Project names are matched case-insensitively on all platforms, matching normal PowerShell command-name ergonomics rather than file-system name rules.

Writes should be atomic: write to a temporary file in the same directory, then replace or move it into place. Invalid or unreadable JSON should produce a clear error and avoid overwriting the existing file.

## Architecture

The feature should be implemented in the management commands area because it is a navigation feature related to `Set-Location`, `Push-Location`, and `Get-Location`.

Components:

- A small project-location store responsible for reading, validating, and atomically writing the JSON file.
- Cmdlets for add, get, enter, and remove operations.
- A terminal picker abstraction used only by `Enter-ProjectLocation` when no name is provided.
- A built-in alias named `pj` that invokes `Enter-ProjectLocation`.

The store should not depend on host UI. Cmdlets should call the store and handle user-facing errors. The picker should not know about JSON storage.

## Error Handling

Expected errors:

- Duplicate project name without `-Force`.
- Invalid project name.
- Path does not exist.
- Path exists but is not a file-system container.
- Persistence file cannot be read or parsed.
- Persistence file cannot be written.
- Picker requested in a non-interactive host.

All failures before a successful jump leave the current location unchanged.

## Testing

Unit tests should cover:

- Add current directory.
- Add explicit path.
- Duplicate add with and without `-Force`.
- Get all projects and get by name.
- Remove existing project.
- Remove missing project.
- Enter existing project.
- Enter missing path leaves current location unchanged.
- Corrupt JSON fails without overwriting the file.
- Non-interactive `Enter-ProjectLocation` without name fails clearly.

The picker itself should be tested through a small abstraction so command behavior can be tested without requiring a real terminal UI.

## Non-Goals

The first version does not include:

- Browser-based project management.
- Automatic project discovery.
- Automatic recording of every directory visited.
- Changing the startup directory of `pwsh`.
- Remote session project switching.
- Non-file-system providers such as Registry or Certificate.
