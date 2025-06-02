# 📖 QuestTextFormatter - Complete Blueprint Documentation

## ✨ New System Advantages

- ✅ **Works perfectly** (no more data table and decorator bullshit)
- ✅ **BlenderPro font integrated** automatically
- ✅ **Quest icons** included
- ✅ **Smooth animations** available
- ✅ **Blueprint compatible** without complexity
- ✅ **Automatic category banners**

## 🎮 Blueprint Usage

### Step 1: Remove the old system

In your Blueprint:
1. Delete all **"Format Quest Description Text"** nodes
2. Delete all **"Setup Quest Description Rich Text"** nodes
3. Delete **"Set Text"** nodes connected to Rich Text Blocks

### Step 2: Modify UMG widgets

In your UMG widget:
1. Delete the **"Rich Text Block"** widget
2. Add a **"Vertical Box"** widget instead
3. Keep the same name (e.g., "QuestDescription")

### Step 3: Use the new functions

## 📋 Available Functions

### 🎯 Main Widget Creation Functions (Recommended)

| Blueprint Function | Description | Parameters | Return Type |
|---|---|---|---|
| **Create Quest Description Widget** | Creates complete quest description | `QuestData`, `FontSize` | `Vertical Box` |
| **Create Quest Rewards Widget** | Displays quest rewards | `QuestData`, `FontSize` | `Vertical Box` |
| **Create Quest Accept Dialog Widget** | Quest acceptance dialog | `QuestData`, `FontSize` | `Vertical Box` |
| **Create Quest Completion Dialog Widget** | Quest completion dialog | `QuestData`, `FontSize` | `Vertical Box` |
| **Create Formatted Text Widget** | Custom formatted text | `FormattedText`, `FontSize` | `Vertical Box` |
| **Create BlenderPro Text Widget** | Simple text with BlenderPro font | `Text`, `FontSize`, `IsBold`, `TextColor` | `Text Block` |

### 📝 Text Parsing & Analysis Functions

| Blueprint Function | Description | Parameters | Return Type |
|---|---|---|---|
| **Parse Formatted Text** | Breaks formatted text into chunks | `FormattedText` | `Array<FTextFormatChunk>` |
| **Get Quest Accept Dialog Text** | Gets raw acceptance dialog text | `QuestData` | `String` |
| **Get Quest Completion Dialog Text** | Gets raw completion dialog text | `QuestData` | `String` |

### 🎨 Styling & Animation Functions

| Blueprint Function | Description | Parameters | Return Type |
|---|---|---|---|
| **Add Fade In Animation** | Adds fade-in animation to widget | `Widget`, `Duration`, `Delay` | `None` |
| **Get BlenderPro Font** | Gets BlenderPro font info | `FontSize`, `IsBold`, `IsItalic` | `Slate Font Info` |
| **Load Icon Texture** | Loads quest icon texture | `IconName` | `Texture 2D` |

### 🔧 Advanced Functions (C++ or Advanced Users)

| Blueprint Function | Description | Parameters | Return Type |
|---|---|---|---|
| **Format Quest Description** | Advanced quest description | `QuestData`, `WidgetTree`, `FontSize` | `Widget` |
| **Format Quest Rewards** | Advanced quest rewards | `QuestData`, `WidgetTree`, `FontSize` | `Widget` |
| **Create Formatted Text** | Advanced formatted text | `FormattedText`, `WidgetTree`, `FontSize` | `Widget` |
| **Create BlenderPro Text Block** | Advanced text block | `Text`, `WidgetTree`, `FontSize`, `IsBold`, `TextColor` | `Text Block` |
| **Format Quest Accept Dialog** | Advanced accept dialog | `QuestData`, `WidgetTree`, `FontSize` | `Widget` |
| **Format Quest Completion Dialog** | Advanced completion dialog | `QuestData`, `WidgetTree`, `FontSize` | `Widget` |

## 🏗️ FTextFormatChunk Structure

The **FTextFormatChunk** struct contains parsed text information:

| Property | Type | Description |
|---|---|---|
| `Text` | `String` | The text content |
| `bIsBold` | `Boolean` | Is text bold |
| `bIsItalic` | `Boolean` | Is text italic |
| `Color` | `Linear Color` | Text color |
| `IconName` | `String` | Icon identifier |
| `bIsIcon` | `Boolean` | Is this chunk an icon |
| `bIsLineBreak` | `Boolean` | Is this a line break |

## 🔗 Usage Examples

### Example 1: Quest Description

```blueprint
[Get Selected Quest Data] ──────► [Create Quest Description Widget]
[Float: 24] ──────────────────► [Font Size]
                                         │
                                         ▼
                                  [Vertical Box Widget] ──► [Add Child to Vertical Box]
                                                                      ▲
                                                               [QuestDescriptionContainer]
```

### Example 2: Text Parsing & Analysis

```blueprint
[String: "<Bold>Hello</Bold> <Red>World</Red>"] ──► [Parse Formatted Text]
                                                           │
                                                           ▼
                                                    [Array of Text Chunks] ──► [For Each Loop]
                                                                                      │
                                                                                      ▼
                                                                               [Process Each Chunk]
```

### Example 3: Custom Animation

```blueprint
[Create BlenderPro Text Widget] ──► [Text Widget] ──► [Add Fade In Animation]
                                                              │
                                                    [Duration: 1.0] ──► [Duration]
                                                    [Delay: 0.5] ────► [Delay]
```

### Example 4: Icon Loading

```blueprint
[String: "icon_contract"] ──► [Load Icon Texture] ──► [Texture] ──► [Set Brush From Texture]
                                                                            ▲
                                                                     [Image Widget]
```

### Example 5: Custom Font Usage

```blueprint
[Float: 18.0] ────► [Get BlenderPro Font] ──► [Font Info] ──► [Set Font]
[Boolean: true] ──► [Is Bold]                                       ▲
[Boolean: false] ─► [Is Italic]                              [Text Block]
```

## 🎨 Supported Formatting Tags

### Text Formatting
- `<Bold>text</Bold>` - **Bold text**
- `<Red>text</Red>` - Red text
- `<Green>text</Green>` - Green text
- `<Yellow>text</Yellow>` - Yellow text
- `<Blue>text</Blue>` - Blue text
- `<White>text</White>` - White text

### Line Breaks
- `\n` - New line
- `<br>` - HTML line break

### Quest Icons
- `<img id="icon_contract"/>` - Contract icon
- `<img id="icon_difficulty_easy"/>` - Easy difficulty icon
- `<img id="icon_difficulty_normal"/>` - Normal difficulty icon
- `<img id="icon_difficulty_medium"/>` - Medium difficulty icon
- `<img id="icon_difficulty_hard"/>` - Hard difficulty icon
- `<img id="icon_difficulty_expert"/>` - Expert difficulty icon
- `<img id="icon_difficulty_elite"/>` - Elite difficulty icon
- `<img id="icon_level_requirement"/>` - Level requirement icon
- `<img id="icon_time_estimate"/>` - Time estimate icon
- `<img id="icon_location"/>` - Location icon
- `<img id="icon_credits"/>` - Credits icon
- `<img id="icon_experience"/>` - Experience icon
- `<img id="icon_reputation"/>` - Reputation icon
- `<img id="icon_item"/>` - Item icon

## 🛠️ UMG Configuration

### Recommended Structure

```
Main Panel (Vertical Box)
├── Quest Title Section
├── Quest Description Container (Vertical Box) ← Replaces Rich Text Block
├── Quest Rewards Container (Vertical Box)
└── Dialog Container (Vertical Box)
```

## ⚡ Recommended Parameters

| Usage | Font Size | Animation Duration |
|---|---|---|
| Quest title | 24.0 | 0.8s |
| Main description | 22.0 | 0.6s |
| Rewards | 20.0 | 0.5s |
| Dialogs | 16.0 | 0.4s |
| Information text | 14.0 | 0.3s |
| Small notes | 12.0 | 0.2s |

## 🎯 Default Colors

- **Quest title**: `FLinearColor(0.9f, 0.8f, 0.2f, 1.0f)` (Gold)
- **Main text**: `FLinearColor::White`
- **Rewards**: `FLinearColor(0.2f, 0.8f, 0.2f, 1.0f)` (Green)
- **Information**: `FLinearColor(0.8f, 0.8f, 0.8f, 1.0f)` (Light gray)

## 🚀 Quick Migration

### Replace your old broken nodes with:

| Old broken node | New node |
|---|---|
| `Format Quest Description Text` → | `Create Quest Description Widget` |
| `Setup Quest Description Rich Text` → | `Create Formatted Text Widget` |
| `Format Quest Rewards` → | `Create Quest Rewards Widget` |
| `Create Rich Text Block` → | `Create BlenderPro Text Widget` |
| `Parse Rich Text` → | `Parse Formatted Text` |

## 🔧 Troubleshooting

### Problem: Widgets don't display
**Solution**: Make sure you add the returned widget to a container (Vertical Box, Panel, etc.)

### Problem: Incorrect font
**Solution**: Ensure the BlenderPro.BlenderPro file exists in `/Game/Widget/Fonts/BlenderPro/`

### Problem: Missing icons
**Solution**: Check that icon textures exist in `/Game/Systems/QuestSystem/Textures/`

### Problem: Animations not working
**Solution**: Make sure the widget is already added to the viewport before calling `Add Fade In Animation`

### Problem: Parse Formatted Text returns empty array
**Solution**: Check your formatting tags are properly closed (e.g., `<Bold>text</Bold>`)

## 📖 Complete Examples

### Basic Quest Display

```blueprint
Event Begin Play
    │
    ▼
[Get Quest Data By ID: "TestQuest"]
    │
    ▼
[Create Quest Description Widget]
    │ Font Size: 22.0
    ▼
[Add Child to Vertical Box: QuestDescriptionContainer]
    │
    ▼
[Add Fade In Animation]
    │ Duration: 0.8, Delay: 0.0
    ▼
[Create Quest Rewards Widget]
    │ Font Size: 20.0
    ▼
[Add Child to Vertical Box: QuestRewardsContainer]
    │
    ▼
[Add Fade In Animation]
    │ Duration: 0.6, Delay: 0.2
```

### Advanced Text Analysis

```blueprint
[String Variable: CustomFormattedText]
    │
    ▼
[Parse Formatted Text]
    │
    ▼
[For Each Loop: Text Chunks]
    │
    ▼
[Switch on Chunk Type]
    ├── [If Is Icon] ──► [Load Icon Texture] ──► [Create Image Widget]
    ├── [If Is Bold] ──► [Create BlenderPro Text Widget: Bold=True]
    └── [If Normal] ───► [Create BlenderPro Text Widget: Bold=False]
```

## 🏗️ Technical Implementation

### Key Features
- **No WidgetTree dependency** - Main functions work directly in Blueprints
- **Advanced WidgetTree functions** - Available for C++ users or advanced scenarios
- **Automatic memory management** - Uses UE5's garbage collection
- **Performance optimized** - Minimal widget creation overhead
- **Modular design** - Each function handles specific use cases
- **Text parsing engine** - Converts formatted strings to structured data

### Required Assets
- **Font**: `/Game/Widget/Fonts/BlenderPro/BlenderPro.BlenderPro`
- **Icons**: `/Game/Systems/QuestSystem/Textures/icon_*.uasset`

## 🧪 Testing Your Implementation

### Basic Test
1. Create a simple string: `"<Bold>Test</Bold> <Red>Quest</Red>"`
2. Use **Parse Formatted Text**
3. Check the returned array has 2 chunks
4. First chunk should have `bIsBold = true`
5. Second chunk should have `Color = Red`

### Widget Test
1. Use **Create Formatted Text Widget** with test string
2. Add to a Vertical Box in your UMG
3. Should display formatted text with BlenderPro font

---

**🎉 Congratulations!** You now have a complete, 100% functional quest text system that completely replaces Unreal Engine's broken RichTextBlock!
