# Simple Color Palette Spec v0.1

> A minimal JSON-based file format for defining color palettes

It uses [extended linear sRGB](#color-space) with optional opacity.

[Learn more ›](https://simplecolorpalette.com)

[**Feedback wanted!**](https://github.com/simple-color-palette/spec/issues)

> [!CAUTION]
> Work in progress.

## Links

- [JSON Schema](schema.json) *(v0.1)*

## JSON Structure

```json
{
	"name": "Favorites",
	"colors": [
		{
			"name": "Hot Pink",
			"components": [1, 0.1274, 0.418]
		},
		{
			"components": [0.592, 0.278, 0.996, 0.9]
		}
	]
}
```

- Palette name is optional.
- Color name is optional.
- Opacity is optional.

## Details

### Top-level Object

- `name`: Optional. String. Name of the palette. If omitted, use the filename without extension. When present, `name` must be non-empty.
- `colors`: Required. Array of color entries. Must contain at least one color. The order is significant and must be preserved.

No other top-level fields are allowed.

### Color Entry

Each color is an object:

- `name`: Optional. String. When present, `name` must be non-empty.
- `components`: Required. Array of 3 or 4 floating-point numbers: `[red, green, blue]` or `[red, green, blue, opacity]`.
	- RGB: **Extended linear sRGB (D65)** (values may be <0 or >1; never clamped).
	- Opacity: Defaults to 1. Clamped to [0, 1] after rounding.

No other color fields are allowed.

### Color Space

**Extended linear sRGB (D65)**: sRGB primaries, D65 white point, linear transfer (identity). "Extended" means values may be <0 or >1; no clipping.

Reference: IEC 61966-2-2 (scRGB, linear variant).

#### Implementation Mapping

- **Apple**: `kCGColorSpaceExtendedLinearSRGB` (not `kCGColorSpaceExtendedSRGB`)
- **ICC/CMS**: ICC v4 profile with sRGB primaries, D65 white, linear TRC
- **Web/CSS**: `color(srgb-linear r g b / a)`; values may be outside [0,1]

### Numeric Precision

Values are defined at a resolution of **1×10⁻⁵**. This is a **quantization rule**, not a formatting requirement:

- **Writers MUST** round all component values to the nearest-even 5-decimal-place step before writing (banker's rounding). For example: 0.123456 → 0.12346, 0.123445 → 0.12344 (nearest even).
- **Readers MUST** accept arbitrary precision (including values with >5 decimal places or in scientific notation like `1e-5`) and **SHOULD** quantize internally to 1×10⁻⁵ resolution for consistency.
- Validators should not reject files with >5 decimal places; this rule applies to writers, not interchange validation.

#### Edge Cases

- Ties round to even: `0.123445 → 0.12344`, `0.123455 → 0.12346`
- Negative values: `-0.000015 → -0.00002`
- Scientific notation accepted: `1e-5` is valid
- Out-of-range opacity example: `[0.5, 0.5, 0.5, 1.2]` is interpreted as `[0.5, 0.5, 0.5, 1.0]` after rounding and clamping

#### Why 5 Decimal Places?

- **HDR precision**: Delivers ~19.9 effective bits over 0–10 range (vs ~16.6 with 4 decimals), exceeding 12-bit perceptually lossless encoding standard (ITU-R Rec. 2100).
- **Industry alignment**: Comparable step size to FP16 workflows, OpenEXR, and ICC profile conversions around typical 0–10 working range.
- **Future-proof**: Handles 12-bit+ displays and 10,000+ nit brightness ranges.
- **Conversion accuracy**: Minimizes rounding error accumulation through color space transformations.
- **Reliable deduplication**: Enables precise color comparison across workflows.

#### Canonicalization Pipeline

When writing color values, follow this order:

1. Perform color calculations at full precision.
2. Round all components (RGB and opacity) to 5 decimal places using banker's rounding.
3. Clamp opacity to [0, 1] range (RGB components are never clamped).
4. Write to JSON.

#### Example (Swift)

```swift
extension Double {
	/**
	Rounds the number to specified decimal places using banker's rounding.

	- Parameter places: Number of decimal places (must be >= 0).
	- Returns: The rounded number.
	*/
	func rounded(toPlaces places: Int) -> Self {
		guard places >= 0 else {
			return self
		}

		let multiplier = pow(10.0, Self(places))
		return (self * multiplier).rounded(.toNearestOrEven) / multiplier
	}
}

let number = 3.14159265359
number.rounded(toPlaces: 5)
//=> 3.14159
```

## Format

### File Extension

`.color-palette`

### File Encoding

UTF-8 without BOM.

### MIME Type

```
application/x.sindresorhus.simple-color-palette+json
```

*(temporary)*

### Uniform Type Identifier

```
com.sindresorhus.simple-color-palette
```

#### Swift example

```swift
extension UTType {
	static var simpleColorPalette: Self { .init(importedAs: "com.sindresorhus.simple-color-palette", conformingTo: .json) }
}
```

#### `Info.plist` example

```xml
<key>UTImportedTypeDeclarations</key>
<array>
	<dict>
		<key>UTTypeIdentifier</key>
		<string>com.sindresorhus.simple-color-palette</string>
		<key>UTTypeDescription</key>
		<string>Simple Color Palette</string>
		<key>UTTypeConformsTo</key>
		<array>
			<string>public.json</string>
		</array>
		<key>UTTypeTagSpecification</key>
		<dict>
			<key>public.filename-extension</key>
			<array>
				<string>color-palette</string>
			</array>
			<key>public.mime-type</key>
			<string>application/x.sindresorhus.simple-color-palette+json</string>
		</dict>
	</dict>
</array>
<key>CFBundleDocumentTypes</key>
<array>
	<dict>
		<key>CFBundleTypeName</key>
		<string>Simple Color Palette</string>
		<key>LSItemContentTypes</key>
		<array>
			<string>com.sindresorhus.simple-color-palette</string>
		</array>
		<key>CFBundleTypeExtensions</key>
		<array>
			<string>color-palette</string>
		</array>
		<key>CFBundleTypeRole</key>
		<string>Viewer</string>
		<key>LSHandlerRank</key>
		<string>Alternate</string>
	</dict>
</array>
```

## FAQ

### Why JSON?

- Human-readable.
- Ubiquitous and supported everywhere.
- Easy to parse in any language.

### Why is the color space hard-coded?

- Simplicity and clarity - no ambiguity about color interpretation.
- Avoiding the complexity of color space conversion and management.
- Color space conversion is better handled at the app level when needed.

### What is gamma correction?

Standard sRGB uses nonlinear encoding to pack color into 8 bits efficiently, shifting precision toward dark tones where the eye is more sensitive. This format uses linear values for accurate color math, requiring gamma correction when converting to/from standard sRGB.

### Why extended linear sRGB (D65)?

- Linear transfer enables accurate color math and blending.
- Extended range (values <0 or >1) represents wide-gamut and HDR colors.
- Well-defined by IEC 61966-2-2 (scRGB); platform-neutral.
- Easy to clamp to standard sRGB when needed.
- Universal: Single working space sufficient for interchange.

### Why linear color?

Linear color ensures mathematically correct operations:
- Color mixing and blending produce perceptually accurate results.
- Alpha compositing works correctly.
- Image processing operations (blur, scale, etc.) avoid artifacts.
- Eliminates gamma correction ambiguity during color calculations.

### Why not Display P3?

Display P3 has wider gamut primaries, but extended linear sRGB (D65) can encode P3 colors via out-of-gamut values (typically negative or >1). A single working space avoids complexity.

### Why no CMYK support?

CMYK is output-dependent and requires device-specific ICC profiles. This format is for authoring and sharing colors, not managing print workflows.

### Does the format support gradients?

No, only flat colors are supported.

## License

The [MIT license](license) applies to the specification itself. You may freely implement this specification without including or referencing the license.
