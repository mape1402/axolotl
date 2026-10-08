# Axolotl

Axolotl is a .NET library project scaffold.

This repository starts with a single packable library project and no implementation code. The layout, release marker, changelog, README, and build workflow are based on the Mule repository, trimmed down for the initial Axolotl package only.

## Package

```bash
dotnet add package Axolotl
```

## Repository Layout

- `src/Axolotl`: the initial Axolotl library project.
- `.github/workflows/build-and-release.yml`: build, package, GitHub release, and NuGet publishing workflow.
- `.release`: release marker used by the workflow.
- `CHANGELOG.md`: release notes source.
- `Directory.Build.props`: shared package and build metadata.

## Build

```bash
dotnet restore Axolotl.sln
dotnet build Axolotl.sln --configuration Release
```

The project currently has no tests and no source files.

## Release

1. Add the release notes under a version heading in `CHANGELOG.md`.
2. Update `.release` with the matching tag, such as `v0.1.0`.
3. Merge the change to `main`.

The workflow creates a release branch, tags the commit, publishes the GitHub release, and pushes the NuGet package when trusted publishing is configured.
