# Multi-App MSIX Subfolder Packaging — Summary of Changes

This document summarizes all changes made to the TwoHeaded solution to package two
WinUI 3 C++ apps (FirstApp and SecondApp) into a single MSIX with each app's files
in its own subfolder.

## Solution Structure

```
TwoHeaded.sln
TwoHeaded/
├── FirstApp/                          
│   ├── FirstApp.vcxproj               
│   ├── FirstApp.vcxproj.filters       
│   ├── App.xaml / .h / .cpp
│   ├── MainWindow.xaml / .h / .cpp / .idl
│   └── app.manifest
├── SecondApp/
│   ├── SecondApp.vcxproj
│   └── (same file set)
└── TwoHeaded (Package)/
    ├── TwoHeaded (Package).wapproj
    └── Package.appxmanifest
```

Resulting MSIX layout:

```
Package root/
├── FirstApp/
│   ├── FirstApp.exe
│   ├── App.xbf          (Debug only; embedded in resources.pri for Release)
│   └── MainWindow.xbf   (Debug only; embedded in resources.pri for Release)
├── SecondApp/
│   ├── SecondApp.exe
│   ├── App.xbf          (Debug only)
│   └── MainWindow.xbf   (Debug only)
├── resources.pri         (shared; contains PRI entries scoped per app)
├── Microsoft.Web.WebView2.Core.dll   (shared dependency at root)
└── (other shared NuGet DLLs at root)
```

---

## 1. XAML `<Link>` Metadata (vcxproj — both apps)

**Problem**: When two apps share a single `resources.pri`, XAML resource URIs collide.
Both apps register `ms-appx:///App.xaml` and `ms-appx:///MainWindow.xaml`, so the
XAML runtime loads the wrong resources for whichever app initializes second.

**Fix**: Add `<Link>` metadata to `ApplicationDefinition` and `Page` items in each
vcxproj. This scopes all generated XAML resource URIs under a per-app prefix.

### FirstApp.vcxproj

```xml
<ApplicationDefinition Include="App.xaml">
  <Link>FirstApp\App.xaml</Link>
</ApplicationDefinition>
<Page Include="MainWindow.xaml">
  <Link>FirstApp\MainWindow.xaml</Link>
</Page>
```

### SecondApp.vcxproj

```xml
<ApplicationDefinition Include="App.xaml">
  <Link>SecondApp\App.xaml</Link>
</ApplicationDefinition>
<Page Include="MainWindow.xaml">
  <Link>SecondApp\MainWindow.xaml</Link>
</Page>
```

**Effect**: The XAML compiler now generates URIs like `ms-appx:///FirstApp/App.xaml`
instead of `ms-appx:///App.xaml`, and PRI entries become `Files/FirstApp/App.xbf`
instead of `Files/App.xbf`. No more collisions.

---

## 2. Additional Include Directories (vcxproj — both apps)

**Problem**: The `<Link>` metadata causes the XAML compiler to place generated `.g.h`
files into a subfolder (e.g., `Generated Files\FirstApp\App.xaml.g.h` instead of
`Generated Files\App.xaml.g.h`), but the source `.cpp` files `#include "App.xaml.g.h"`
without a subfolder prefix.

**Fix**: Add the subfolder to `AdditionalIncludeDirectories` so the compiler can find
the generated headers.

### FirstApp.vcxproj

```xml
<ItemDefinitionGroup>
  <ClCompile>
    <AdditionalIncludeDirectories>
      %(AdditionalIncludeDirectories);$(ProjectDir)Generated Files\FirstApp
    </AdditionalIncludeDirectories>
  </ClCompile>
</ItemDefinitionGroup>
```

### SecondApp.vcxproj

```xml
<AdditionalIncludeDirectories>
  %(AdditionalIncludeDirectories);$(ProjectDir)Generated Files\SecondApp
</AdditionalIncludeDirectories>
```

---

## 3. Package.appxmanifest — Two Application Entries

**Change**: Added a second `<Application>` element so both apps are registered in
the MSIX and launchable from the Start menu.

```xml
<Applications>
  <Application Id="App"
    Executable="FirstApp\FirstApp.exe"
    EntryPoint="Windows.FullTrustApplication">
    <uap:VisualElements DisplayName="FirstApp" ... />
  </Application>
  <Application Id="SecondApp"
    Executable="SecondApp\SecondApp.exe"
    EntryPoint="Windows.FullTrustApplication">
    <uap:VisualElements DisplayName="Second App" ... />
  </Application>
</Applications>
```

Key detail: `Executable` paths include the subfolder prefix (`FirstApp\FirstApp.exe`).

---

## 4. Wapproj — Custom MSBuild Targets

Two custom targets in `TwoHeaded (Package).wapproj` handle subfolder placement and
XBF deduplication.

### 4a. `_PrefixProjectOutputsWithSubfolder` (AfterTargets="_ConvertItems")

This target runs after the packaging system collects all project outputs and does
the following:

1. **Prefixes `TargetPath` and `CopyToTargetPath`** on three MSBuild item lists to
   place each app's files in its subfolder:
   - `WapProjPackageFile` — controls file copy and map generation
   - `UploadWapProjPackageFile` — controls upload package layout
   - `PackagingOutputs` — used by `RemovePayloadDuplicates` for collision checks

   Items are matched by `%(SourceProject)` metadata (set to the project name by the
   wapproj system) and filtered to exclude shared NuGet packages.

2. **Removes `.xbf` items in Release**: In Release builds, XBF files are embedded in
   `resources.pri` as `EmbeddedData`, so loose XBF files are redundant. The target
   removes them from `WapProjPackageFile`, `UploadWapProjPackageFile`, and
   `PackagingOutputs` when `$(Configuration) == Release`.

3. **Rebuilds derived item lists** (`_FileToCopy`, `_UploadFileToCopy`,
   `AppxPackagePayload`, `AppxUploadPackagePayload`) from the modified source lists.

4. **Updates `$(EntryPointExe)`** to include the subfolder prefix.

**Note on XBF items**: With the `<Link>` metadata in place, XBF `TargetPath` values
already include the subfolder (e.g., `FirstApp\App.xbf`), so no additional prefixing
is needed for XBF files. Only exe/dll/pri items need the subfolder prefix added here.

### 4b. `_CleanupRootDuplicatesAndAddSubfolderPri` (AfterTargets="_UpdateMainPackageFileMap")

This target filters XBF entries from the MakeAppx map files after they are generated
but before MakeAppx packs the MSIX. It writes and executes a PowerShell script that:

- **Debug**: Removes only root-level XBF entries (those without a backslash in the
  target path) since they are duplicates of the subfolder copies.
- **Release**: Removes ALL XBF entries since they are embedded in `resources.pri`.

It processes both `main.map.txt` (used by Debug/Bundle builds) and `package.map.txt`
(used by Release non-bundle builds).

### 4c. Project References with `<PackageFolder>` Metadata

```xml
<ProjectReference Include="..\FirstApp\FirstApp.vcxproj">
  <PackageFolder>FirstApp</PackageFolder>
</ProjectReference>
<ProjectReference Include="..\SecondApp\SecondApp.vcxproj">
  <PackageFolder>SecondApp</PackageFolder>
</ProjectReference>
```

The `<PackageFolder>` metadata is used by the wapproj system to set the
`%(SourceProject)` metadata on packaging items, which the custom targets use to
determine which subfolder each file belongs to.

---

## Key Technical Insights

1. **`<Link>` metadata is the linchpin**: It causes three cascading effects that
   solve the URI collision problem — scoped XAML URIs, scoped PRI entries, and
   scoped generated file paths.

2. **Debug vs Release XBF handling differs**: Debug uses loose `.xbf` files
   referenced by PRI as `type="Path"`. Release embeds them in `resources.pri` as
   `type="EmbeddedData"`. The wapproj must handle both cases.

3. **Three MSBuild item lists must stay in sync**: `WapProjPackageFile`,
   `UploadWapProjPackageFile`, and `PackagingOutputs` all need consistent subfolder
   prefixing, plus the derived lists must be rebuilt from them.

4. **Two map files exist**: `main.map.txt` (Debug/Bundle) and `package.map.txt`
   (Release non-bundle). Both need XBF filtering.

5. **Shared dependencies stay at root**: NuGet package DLLs (e.g., WebView2) are
   excluded from subfolder prefixing so they remain at the package root, shared by
   both apps.
