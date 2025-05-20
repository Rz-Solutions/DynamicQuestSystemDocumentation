# Rich Text Support in Quest Descriptions

The Dynamic Quest System now supports rich text formatting in quest descriptions and objective descriptions. This allows you to create more visually appealing and informative quest text with formatting such as bold, italics, colors, and line breaks.

## Basic Formatting

You can use the following tags in your description text:

- **Bold Text**: `<bold>Bold text here</bold>`
- **Italic Text**: `<italic>Italicized text here</italic>`
- **Colored Text**: `<color=#FF0000>Red text</color>` (Use hex color codes)
- **Line Breaks**: Just press Enter in the description field, or use `<br>` tags

## Examples

Here's an example of a formatted quest description:

```
<bold>Retrieve the Lost Drone</bold>

Your mission is to find the <color=#FFCC00>Drone</color> in the ruins of <italic>in front of Djibouti</italic>.

<color=#FF0000>WARNING:</color> The ruins are infested with sanglines!
```

## Advanced Formatting

For more advanced formatting, you can combine tags:

- **Bold and Colored**: `<bold><color=#00FF00>Important objective</color></bold>`
- **Italic and Colored**: `<italic><color=#0088FF>Special location</color></italic>`

## Compatibility

Rich text is displayed correctly in:
- Quest logs
- Quest objectives
- Quest trackers
- Quest notifications

## Technical Notes

- The text is automatically wrapped in `<RichText>` tags if not already present
- Line breaks are automatically converted to `<br>` tags
- If you're creating UI widgets that display quest descriptions, use RichTextBlock instead of TextBlock

## Blueprint Function Library

You can use the `UQuestRichTextHelper` class in Blueprints to format text:

- **FormatQuestDescriptionText**: Formats quest description text with rich text support
- **FormatObjectiveDescriptionText**: Formats objective description text with rich text support
- **SetupQuestDescriptionRichText**: Configures a RichTextBlock for optimal quest description display

---

For more information on Unreal Engine's rich text support, see the [official documentation](https://docs.unrealengine.com/5.3/en-US/umg-rich-text-blocks-in-unreal-engine/).