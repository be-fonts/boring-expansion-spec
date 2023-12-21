# `VARC` table: Variable Composites / Components

This table encodes *variable-composite* glyphs in a way that can be combined
with any of `glyf`/`gvar` or `CFF2`. Since the primary purpose of
variable-composites is to make font files (especially the CJK fonts) smaller,
several new data-structures are introduced to achieve more compression compared
to an equivalent non-variable-composite font.


## Data Structures

The following foundational data-structures are used in this able:

- `VarInt32` is a variable-length encoding of `uint32` values, which uses 1 to
  5 bytes depending on the magnitude of the value being encoded. It borrows
  ideas from the UTF-8 encoding, but is more efficient because it does not have
  some of the random-access properties of UTF-8 encoding.

- `TupleValues` is borrowed from the `TupleVariationStore` [Packed
  Deltas](https://learn.microsoft.com/en-us/typography/opentype/spec/otvarcommonformats#packed-deltas)
  with minor modification to allow storing 32-bit values.

- `CFF2IndexOf` is simply a CFF2-style `Index` structure containing data of a
  particular type.  CFF2 `Index` is defined
  [here](https://learn.microsoft.com/en-us/typography/opentype/spec/cff2#5-index-data).
  The purpose of `CFF2IndexOf` is to efficiently store a list of variable-sized
  data, for example glyph records.

- `MultiItemVariationStore`: This is a new data-structure. It is a hybrid
  between `ItemVariationStore`, and `TupleVariationStore`, borrowing ideas from
  both and improving upon them for more efficient storage of variations of
  *tuples* of numbers.


### `VarInt32`

To be added to [The OpenType Font File Data
Types](https://learn.microsoft.com/en-us/typography/opentype/spec/otff#data-types)

A `VarInt32` is a variable-length encoding of a `uint32`. Like the UTF-8
encoding, it encodes the number of subsequent bytes as top bits of the first
byte. Unlike UTF-8, the subsequent bytes use the full 8 bits for storage,
instead of 6 in UTF-8.

*TODO:* Add table of encoding bytes.

Here is Python code to read and write `VarInt32` values:

```python
def _readVarInt32(data, i):
    """Read a variable-length number from data starting at index i.

    Return the number and the next index.
    """

    b0 = data[i]
    if b0 < 0x80:
        return b0, i + 1
    elif b0 < 0xC0:
        return (b0 - 0x80) << 8 | data[i + 1], i + 2
    elif b0 < 0xE0:
        return (b0 - 0xC0) << 16 | data[i + 1] << 8 | data[i + 2], i + 3
    elif b0 < 0xF0:
        return (b0 - 0xE0) << 24 | data[i + 1] << 16 | data[i + 2] << 8 | data[
            i + 3
        ], i + 4
    else:
        return (b0 - 0xF0) << 32 | data[i + 1] << 24 | data[i + 2] << 16 | data[
            i + 3
        ] << 8 | data[i + 4], i + 5
```
```python
def _writeVarInt32(v):
    """Write a variable-length number.

    Return the data.
    """
    if v < 0x80:
        return struct.pack(">B", v)
    elif v < 0x4000:
        return struct.pack(">H", (v | 0x8000))
    elif v < 0x200000:
        return struct.pack(">L", (v | 0xC00000))[1:]
    elif v < 0x10000000:
        return struct.pack(">L", (v | 0xE0000000))
    else:
        return struct.pack(">B", 0xF0) + struct.pack(">L", v)
```

### `TupleValues`

`TupleValues` is similar to the `TupleVariationStore` [Packed
Deltas](https://learn.microsoft.com/en-us/typography/opentype/spec/otvarcommonformats#packed-deltas)
with a minor modification: if the top two bites of the control byte
(`DELTAS_ARE_ZERO` and `DELTAS_ARE_WORDS`) are both set, then the following
values are 32-bit. That difference should be incorporated in the Packed Deltas
section of `TupleVariationStore`, and is backwards-compatible because the two
top bits can currently never be set at the same time.

`TupleValues` can be used in two different ways:
- When the number of values to be decoded is known in advance, decoding stops
  when the needed number of values are decoded.
- When the number of values to be decoded is _not_ known in advance, but the
  number of encoded bytes is known, then values are decoded until the bytes are
  depleted.


### `CFF2IndexOf`

`CFF2IndexOf` is simply a CFF2-style `Index` structure containing data of a particular type.
CFF2 `Index` is defined
[here](https://learn.microsoft.com/en-us/typography/opentype/spec/cff2#5-index-data).
The purpose of `CFF2IndexOf` is to efficiently store a list of variable-sized
data, for example glyph records, or other data.

Note that the `count` field of CFF2 `Index` structure is `uint32`, unlike the
count field of CFF Index structure.

### `MultiItemVariationStore`

To be added to [OpenType Font Variations Common Table
Formats](https://learn.microsoft.com/en-us/typography/opentype/spec/otvarcommonformats).

A `MultiItemVariationStore` is a new data-structure. It is a hybrid between
`ItemVariationStore`, and `TupleVariationStore`, borrowing ideas from both and
improving upon them for more efficient storage of variations of *tuples* of
numbers.

Like `ItemVariationStore`, entries are addressed using a 32-bit `VarIdx`,
with the top 16 bits called "outer" index, and lower 16 bits called the "inner"
index.

Whereas the `ItemVariationStore` stores deltas for a single scalar value for
each `VarIdx`, the `MultiItemVariationStore` stores deltas for a tuple for each
`VarIdx`.

Compared to `TupleVariationStore`, the `MultiItemVariationStore` is optimized
for smaller tuples and allows tuple-sharing, which is important for its
efficiency over the `TupleVariationStore`. It also does not have some of the
limitations that `TupleVariationStore` has, like the total size of an entry
being limited to 64kb.

The top-level header of a `MultiItemVariationStore` is:
```
struct MultiItemVariationStore
{
  uint16 format; // Set to 1
  LOffsetTo<SparseVariationRegionList> variationRegionListOffset;
  uint16_t itemVariationDataCount
  Offset32To<MultiItemVariationData> itemVariationDataOffsets[itemVariationDataCount];
};
```

*TODO* Finish.

```
struct MultiItemVariationData
{
  uint16 Format; // 1
  uint16 VarRegionCount;
  uint16 VarRegionIndex[VarRegionCount]
  CFF2IndexOf<TupleValues>
};
```


## Variable Composite Description

A Variable Composite record is a concatenation of Variable Component records. Variable Component records have varying sizes.
```
struct VarCompositeGlyph
{
  VarComponent components[];
};
```

## Variable Component Record

A Variable Component record encodes one component's glyph index, variations location, and transformation in a variable-sized and efficient manner.

```
struct VarComponent
{
  uint16 flags;
  _variable_
}
```

| type | name | notes |
|-|-|-|
| uint16 | flags | See below. |
| GlyphID16 or GlyphID24 | gid | This is a GlyphID16 if bit 2 of `flags` is clear, else GlyphID24. |
| VarInt32 | axisIndicesIndex | Optional, only present if bit 3 of `flags` is set. |
| TupleValues | axisValues | The axis value for each axis, variable sized. |
| VarInt32 | axisValuesVarIndex | Optional, only present if bit 4 of `flags` is set. |
| VarInt32 | transformVarIndex | Optional, only present if bit 5 of `flags` is set. |
| FWORD | TranslateX | Optional, only present if bit 6 of `flags` is set. |
| FWORD |  TranslateY | Optional, only present if bit 7 of `flags` is set. |
| F4DOT12 | Rotation | Optional, only present if bit 8 of `flags` is set. Clockwise. |
| F6DOT10 | ScaleX | Optional, only present if bit 9 of `flags` is set. |
| F6DOT10 | ScaleY | Optional, only present if bit 10 of `flags` is set. |
| F4DOT12 | SkewX | Optional, only present if bit 11 of `flags` is set. Clockwise. |
| F4DOT12 | SkewY | Optional, only present if bit 12 of `flags` is set. Clockwise. |
| FWORD | TCenterX | Optional, only present if bit 13 of `flags` is set. |
| FWORD |  TCenterY | Optional, only present if bit 14 of `flags` is set. |

### Variable Component Flags

| bit number | meaning |
|-|-|
| 0 | use my metrics |
| 1 | reset unspecified axes |
| 2 | gid is 24 bit |
| 3 | have axes |
| 4 | axis values have variation |
| 5 | transformation has variation |
| 6 | have TranslateX |
| 7 | have TranslateY |
| 8 | have Rotation |
| 9 | have ScaleX |
| 10 | have ScaleY |
| 11 | have SkewX |
| 12 | have SkewY |
| 13 | have TCenterX |
| 14 | have TCenterY |
| 15 | Reserved. Set to 0 |

### Variable Component Transformation

The transformation data consists of individual optional fields, which can be
used to construct a transformation matrix.

Transformation fields:

| name | default value |
|-|-|
| TranslateX | 0 |
| TranslateY | 0 |
| Rotation | 0 |
| ScaleX | 1 |
| ScaleY | ScaleX |
| SkewX | 0 |
| SkewY | 0 |
| TCenterX | 0 |
| TCenterY | 0 |

The `TCenterX` and `TCenterY` values represent the “center of transformation”.

Details of how to build a transformation matrix, as pseudo-Python code:

```python
# Using fontTools.misc.transform.Transform
t = Transform()  # Identity
t = t.translate(TranslateX + TCenterX, TranslateY + TCenterY)
t = t.rotate(Rotation * math.pi)
t = t.scale(ScaleX, ScaleY)
t = t.skew(-SkewX * math.pi, SkewY * math.pi)
t = t.translate(-TCenterX, -TCenterY)
```

## `VARC` table
```
struct VARC
{
  uint16_t major; // 1
  uint16_t minor; // 0
  Offset32To<Coverage> coverage;
  Offset32To<MultiItemVariationStore>
  Offset32To<CFF2IndexOf<VarCompositeGlyph>> glyphRecords;
};

struct VarCompositeGlyphRecord
{
  VarComponentGlyphRecord[] components;
};
```

This design uses two new datastrucure: `MultiItemVariationStore`, and
`CFF2IndexOf`. Let's look at those

- `CFF2IndexOf` is simply `CFFIndex` containing data of a particular type.
  `CFF2Index` is defined here:
  https://learn.microsoft.com/en-us/typography/opentype/spec/cff2#5-index-data

- `MultiIteVariationStore`: This is a new datastructure. It's a hybrid between
  `ItemVariationStore`, and `TupleVariationStore`, borring ideas (and
  data-structures) from both. Defined below:


## Processing

The component glyphs to be loaded use the coordinate values specified (with any
variations applied if present). For any unspecified axis, the value used
depends on flag bit 13. If the flag is set, then the normalized value zero is
used. If the flag is clear the axis values from current glyph being processed
(which itself might recursively come from the font or its own parent glyphs)
are used.  For example, if the font variations have `wght`=.25 (normalized),
and current glyph being processed is using `wght`=.5 because it was referenced
from another VarComposite glyph itself, when referring to a component that does
_not_ specify the `wght` axis, if flag bit 13 is set, then the value of
`wght`=0 (default) will be used. If flag bit 13 is clear, `wght`=.5 (from
current glyph) will be used.

The component location and transform can vary. These variations are stored in
the `MultiItemVariationStore` data-structure. The variations for location are
referred to by the `axisValuesVarIndex` member of a component if any, and
variations for transform are referred to by the `transformVarIndex` if any. For
transform variations, only those fields specified and as such encoded as per
the flags have variations.

**Note:** While it is the (undocumented?) behavior of the `glyf` table that
glyphs loaded are shifted to align their LSB to that specified in the `hmtx`
table, much like regular Composite glyphs, this does not apply to component
glyphs being loaded as part of a variable-composite glyph.

**Note:** A static (non-variable) font that uses the `VARC` table, _would not_
have `fvar` / `avar` tables but _would_ have the `gvar` table in a
TrueType-flavored font, or `CFF2` variations in a CFF-flavored font. This is
because the components themselves store their variables in the classic way. The
exception to this situation is a font with `VARC` table where does NOT vary the
component locations and transforms, and does not encode any location for the
components either. In practice, such a font would just use the `VARC` table
like classic components.