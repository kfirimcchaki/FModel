# FModel Asset Editor — Integration Guide

## New Files Added

These files are already in the repo and ready to use:

| File | Purpose |
|------|---------|
| `CUE4Parse/CUE4Parse/UE4/Assets/Writer/AssetBinaryEditor.cs` | Core engine: raw byte extraction, hex search/replace, write-back to pak/ucas, JSON patch export/import |
| `CUE4Parse/CUE4Parse/UE4/Assets/Writer/PropertyEditor.cs` | Structured property-level editing (bool, int, float, string, enum, struct, etc.) |
| `FModel/ViewModels/AssetEditor/AssetEditorViewModel.cs` | ViewModel bridging UI with the editing engine |
| `FModel/ViewModels/Commands/EditAssetCommand.cs` | Context menu command to launch the editor |
| `FModel/Views/AssetEditor/AssetEditorWindow.xaml` | Editor UI (hex editor, property editor, log tabs) |
| `FModel/Views/AssetEditor/AssetEditorWindow.xaml.cs` | Code-behind for the editor window |

## Existing Files to Modify

### 1. `FModel/ViewModels/ApplicationViewModel.cs`

Add the `EditAssetCommand` property:

```csharp
// Find the line where other commands are declared (around line 30-50)
// Add:
public EditAssetCommand EditAssetCommand { get; }

// In the constructor, add:
EditAssetCommand = new EditAssetCommand(this);
```

### 2. `FModel/Views/Resources/Controls/ContextMenus/FileContextMenu.xaml`

Add the "Edit Asset" menu item. Insert this block **before** the first `<Separator />`:

```xml
<MenuItem Header="✏️ Edit Asset" Command="{Binding EditAssetCommand}">
    <MenuItem.CommandParameter>
        <MultiBinding Converter="{x:Static converters:MultiParameterConverter.Instance}">
            <Binding Path="PlacementTarget.SelectedItems"
                     RelativeSource="{RelativeSource AncestorType=ContextMenu}" />
        </MultiBinding>
    </MenuItem.CommandParameter>
    <MenuItem.IsEnabled>
        <Binding Path="PlacementTarget.SelectedItems"
                 RelativeSource="{RelativeSource AncestorType=ContextMenu}">
            <Binding.Converter>
                <converters:AnyItemMeetsConditionConverter>
                    <converters:AnyItemMeetsConditionConverter.Conditions>
                        <converters:ItemIsUePackageCondition />
                    </converters:AnyItemMeetsConditionConverter.Conditions>
                </converters:AnyItemMeetsConditionConverter>
            </Binding.Converter>
        </Binding>
    </MenuItem.IsEnabled>
    <MenuItem.Icon>
        <Viewbox Width="16" Height="16">
            <Canvas Width="24" Height="24">
                <Path Fill="{DynamicResource {x:Static adonisUi:Brushes.ForegroundBrush}}"
                      Data="M3 17.25V21h3.75L17.81 9.94l-3.75-3.75L3 17.25zM20.71 7.04c.39-.39.39-1.02 0-1.41l-2.34-2.34c-.39-.39-1.02-.39-1.41 0l-1.83 1.83 3.75 3.75 1.83-1.83z" />
            </Canvas>
        </Viewbox>
    </MenuItem.Icon>
</MenuItem>
```

### 3. `FModel/Views/Resources/Icons.xaml`

Add the edit icon geometry (optional, you can also use inline Data as shown above):

```xml
<Geometry x:Key="EditAssetIcon">M3 17.25V21h3.75L17.81 9.94l-3.75-3.75L3 17.25zM20.71 7.04c.39-.39.39-1.02 0-1.41l-2.34-2.34c-.39-.39-1.02-.39-1.41 0l-1.83 1.83 3.75 3.75 1.83-1.83z</Geometry>
```

---

## How It Works

### Architecture Overview

```
User right-clicks asset → "Edit Asset"
        │
        ▼
EditAssetCommand.Execute()
        │
        ├── Extracts raw bytes via CUE4Parse (entry.Read())
        ├── Tries to parse Package for structured properties
        │
        ▼
AssetEditorWindow opens with 3 tabs:
        │
        ├── 🔧 Property Editor (structured editing)
        │       • Lists all properties with types and values
        │       • Double-click to edit (same-size in-place modification)
        │       • Changes applied to raw binary via PropertyEditor
        │
        ├── 🔢 Hex Editor (raw byte editing)
        │       • Full hex dump view
        │       • Search hex patterns (with ?? wildcards)
        │       • Replace hex patterns (single or all occurrences)
        │       • Changes applied directly to byte array
        │
        └── 📋 Log (activity log)

After editing, the user can:
        │
        ├── 💾 Save to Container
        │       • Writes modified bytes directly into .pak or .ucas
        │       • Creates backup of original container file
        │       • Only works for uncompressed, unencrypted entries
        │       • Same-size data only (no repack needed)
        │
        ├── 📄 Export Patch JSON
        │       • Diffs original vs modified bytes
        │       • Exports JSON with byte-level patches
        │       • Includes search context for fuzzy matching
        │       • Can be re-applied to fresh game files
        │
        ├── 📦 Export Modified Asset
        │       • Saves the full modified .uasset to disk
        │       • Preserves the game's folder structure
        │
        └── 📥 Import Patch JSON
                • Load a previously exported patch
                • Apply to current asset (with verification)
                • Supports search-context fuzzy matching
```

### JSON Patch Format

```json
{
  "version": 1,
  "description": "Asset patches exported from FModel Asset Editor",
  "createdAt": "2026-05-27T...",
  "entries": [
    {
      "assetPath": "FortniteGame/Content/Creative/CreativeBetaPermissions.uasset",
      "containerFile": "pakchunk0-WindowsClient.pak",
      "containerType": "pak",
      "assetOffset": 12345678,
      "assetSize": 4096,
      "originalHash": "abc123...",
      "modifiedHash": "def456...",
      "patches": [
        {
          "offsetInAsset": 256,
          "originalBytes": "00 00 80 3F",
          "replacementBytes": "00 00 00 40",
          "searchContext": "42 6F 6F 6C 00 00 80 3F 00 00 00 00",
          "contextOffset": 6,
          "label": "Changed float from 1.0 to 2.0"
        }
      ]
    }
  ]
}
```

### Limitations

1. **Same-size edits only** for direct container write-back. If your edit changes the data size, export the modified asset or use the JSON patch.

2. **Compressed/encrypted entries** cannot be written back in-place. The tool will tell you when this is the case. Use the JSON patch export + a repacking tool instead.

3. **String edits** in the property editor must maintain the same byte length. Pad with null bytes or use the hex editor for different-length strings.

4. **FName references** (property names, class names) cannot be edited through the property editor because they reference the package's name table. Use hex editing for these.

5. **IoStore (.ucas) entries** with compression require repacking the entire container. For these, export the modified asset and use a pak/ucas repacker.

---

## Building

The new files use:
- `CUE4Parse` (already a dependency)
- `Newtonsoft.Json` (already a dependency)
- `AdonisUI` (already a dependency)
- Standard .NET/WPF (already in project)

No additional NuGet packages needed. Just build the solution normally.
