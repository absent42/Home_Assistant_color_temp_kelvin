# Home Assistant Color Temp Migration Tool

A single-page web tool that converts deprecated Home Assistant light color temperature YAML to the new format introduced in HA 2026.3.

Download and run locally, or use the online version here: https://absent42.github.io/Home_Assistant_color_temp_kelvin/

## Background

Home Assistant 2026.3 ([details](https://developers.home-assistant.io/blog/2026/02/23/remove-deprecate-light-features/)) removed several light entity attributes and service call parameters:

| Removed | Replacement |
|---------|-------------|
| `color_temp` (mireds) service parameter | `color_temp_kelvin` (Kelvin) |
| `kelvin` service parameter | `color_temp_kelvin` |
| `color_temp` state attribute | `color_temp_kelvin` |
| `min_mireds` state attribute | `min_color_temp_kelvin` |
| `max_mireds` state attribute | `max_color_temp_kelvin` |

Any automations, scripts, or scenes using the old parameters will break after upgrading.

## How to Use

1. Open `index.html` in any modern browser (or visit the hosted version)
2. Paste your YAML into the **Input** panel (whole files or snippets both work)
3. Click **Convert**
4. Review the highlighted output and summary
5. Click **Copy Result** to copy the converted YAML to your clipboard
6. Paste it back into your HA configuration

You can also click **Load Example** to see a demo with all conversion types.

## What It Converts

### Literal mired values (green - auto-converted)

```yaml
# Before
color_temp: 370

# After
color_temp_kelvin: 2703
```

The math: `round(1,000,000 / mireds)`. So `370 mireds = 2703 Kelvin`.

### Kelvin parameter rename (green - auto-converted)

```yaml
# Before
kelvin: 3000

# After
color_temp_kelvin: 3000
```

Just a rename, no math needed.

### Template values (yellow - review recommended)

```yaml
# Before
color_temp: "{{ state_attr('light.desk', 'color_temp') }}"

# After
color_temp_kelvin: "{{ state_attr('light.desk', 'color_temp_kelvin') }}"
```

The key is renamed and any attribute references inside the template are also updated. Flagged for review because if your template returns a value you know to be in mireds (e.g. from an `input_number` you set up in mireds), you will need to add conversion math manually.

### Dynamic/unresolvable values (red - manual fix needed)

```yaml
# Before
color_temp: !input user_color_temp

# After
color_temp_kelvin: !input user_color_temp  # MANUAL FIX NEEDED: ...
```

The key is renamed but you need to verify the value provides Kelvin. A comment is added with guidance.

### Attribute reads in templates (yellow - review recommended)

```yaml
# Before
value_template: "{{ state_attr('light.desk', 'color_temp') > 300 }}"

# After (with auto-convert comparisons enabled)
value_template: "{{ state_attr('light.desk', 'color_temp_kelvin') > 3333 }}"
```

Attribute references (`'color_temp'`, `'min_mireds'`, `'max_mireds'`, `attributes.color_temp`, etc.) are renamed to their new equivalents.

## Options

### Auto-convert mired values in comparisons

Enabled by default. When a line contains an attribute rename and also has a numeric comparison (e.g. `> 300`), the tool will convert the number from mireds to Kelvin if it falls within the typical mired range (100-600).

Uncheck this if your comparisons use values that are already in Kelvin or are not color temperature values.

## Pitfalls and Limitations

### Values outside the 100-600 mired range are not auto-converted

The auto-convert comparisons feature only converts numbers between 100 and 600 (the typical mired range for smart bulbs). If you have comparison values outside this range, they will not be converted automatically. Review these manually.

### Comments in YAML are preserved but not converted

If a comment mentions a mired value (e.g. `# warm light is 370 mireds`), the tool will not update it. Comments starting with `#` at the beginning of a line are skipped entirely.

### Template expressions returning mireds need manual attention

If you have a template like `color_temp: "{{ states('input_number.my_temp') }}"` and the `input_number` stores a mired value, simply renaming to `color_temp_kelvin` will pass the wrong unit. You need to either:

- Change the `input_number` to store Kelvin values, or
- Add conversion math: `"{{ (1000000 / states('input_number.my_temp') | float) | round(0) | int }}"`

### Multi-line templates are not supported

The tool works line-by-line. Jinja2 templates that span multiple lines (using `>` or `|` YAML block scalars) will not be detected. You will need to convert these manually.

### The tool only handles light color temperature

It does not convert climate entity temperature references or any other use of `color_temp` outside of the light domain. If you have custom integrations or other uses of these attribute names, review those separately.

### `!input` and `!include` values cannot be statically resolved

These are flagged red with a comment. You need to trace where the value comes from and ensure it provides Kelvin.

### False positives on attribute names

If you have a custom sensor or entity that happens to use `color_temp`, `min_mireds`, or `max_mireds` as attribute names for non-light purposes, the tool may incorrectly rename them. Review yellow-flagged changes carefully.

## Hosting

The tool is a single `index.html` file with no external dependencies. You can:

- Open it directly in your browser from your filesystem
- Host it on GitHub Pages
- Put it on any static file server

It works completely offline after the page loads.

## Technical Details

- Pure HTML/CSS/JS, no frameworks or libraries
- Line-by-line regex-based conversion (not full YAML parsing) to preserve formatting, comments, and YAML anchors/aliases
- Dark theme matching the Home Assistant aesthetic
- Responsive layout (stacks vertically on mobile)
