# Panorama Extension Specification

- **Title:** Panorama
- **Identifier:** <https://360-geo.github.io/stac-panorama/v1.0.0/schema.json>
- **Field Name Prefix:** pano
- **Scope:** Item, Collection
- **Extension [Maturity Classification](https://github.com/radiantearth/stac-spec/tree/master/extensions/README.md#extension-maturity):** Proposal
- **Owner**: @georgeboot

This document explains the Panorama Extension to the [SpatioTemporal Asset Catalog](https://github.com/radiantearth/stac-spec) (STAC)
specification. It describes 360° panoramas: which projection the image uses, and how it is oriented in the world.

No other extension does this. [Perspective Imagery](https://github.com/stac-extensions/perspective-imagery) (`pers:`) models frame
cameras. [View](https://github.com/stac-extensions/view) has `view:azimuth`, a true-north bearing, and no pitch or roll. Street-level
catalogs such as [Panoramax](https://panoramax.fr) mark a panorama by setting `pers:interior_orientation.field_of_view` to 360 and
publish a heading in `view:azimuth`. That tells a client the image is wide, but not how it is laid out. It also leaves open which
pixel the heading applies to, and a panorama faces every direction at once.

This extension adds:

- an explicit projection type, `pano:projection_type`;
- the orientation of the panorama as yaw, pitch and roll: `pano:yaw`, `pano:pitch` and `pano:roll`. These mean the same as the
  GPano `PoseHeadingDegrees`, `PosePitchDegrees` and `PoseRollDegrees` fields, specified down to the pixel.

It builds on `pers:`. The angles are measured in the frame of `pers:crs` and describe the same rotation as `pers:rotation_matrix`.
It can also be used without `pers:`, in which case north is true north. It answers
[perspective-imagery#12](https://github.com/stac-extensions/perspective-imagery/issues/12).

- Examples:
  - [Georizon Item](examples/item-georizon.json): a surveyed panorama in a projected CRS (RD New + NAP), with `pers:` fields
  - [Panoramax Item](examples/item-panoramax.json): a panorama in the shape Panoramax publishes, with its GPano angles carried over
  - [Collection](examples/collection.json): summaries and item asset definitions
- [JSON Schema](json-schema/schema.json)
- [Changelog](./CHANGELOG.md)

## Fields

The fields in the table below can be used in these parts of STAC documents:

- [ ] Catalogs
- [ ] Collections
- [x] Item Properties (incl. Summaries in Collections)
- [x] Assets (for both Collections and Items, incl. Item Asset Definitions in Collections and Asset Templates)
- [ ] Links
- [ ] Bands

| Field Name           | Type   | Description |
| -------------------- | ------ | ----------- |
| pano:projection_type | string | **REQUIRED** in Item Properties. How the image maps directions to pixels. One of: `equirectangular`. |
| pano:yaw             | number | Heading of the [reference direction](#the-reference-direction) in degrees, clockwise from north, `[0, 360)`. |
| pano:pitch           | number | Elevation of the reference direction above the horizontal in degrees, `[-90, 90]`. Positive looks up. |
| pano:roll            | number | Rotation about the reference direction in degrees, `(-180, 180]`. Positive when the right-hand edge tilts down. |

`(0, 0, 0)` means the centre of the image looks north at the horizon, with the image upright.

### Additional Field Information

#### pano:projection_type

The only value in this version is `equirectangular`: the full sphere, 360° across and 180° from top to bottom. See
[Equirectangular images](#equirectangular-images).

This field is required in Item Properties because it is what marks an Item as a panorama, and what clients filter on. It replaces
the convention of setting `pers:interior_orientation.field_of_view` to 360. Producers MAY keep setting that field for older clients,
but `pano:projection_type` takes precedence.

This version only describes panoramas that cover the full sphere. Cropped panoramas are listed under [Future work](#future-work).

#### Orientation fields

`pano:yaw`, `pano:pitch` and `pano:roll` are each optional, and producers SHOULD give all three when they are known.

- A missing `pano:pitch` or `pano:roll` is unknown. Clients commonly assume 0, which means level, as GPano does.
- A missing `pano:yaw` means the heading is unknown and the panorama cannot be aligned to north.
- At `pano:pitch` of exactly ±90°, yaw and roll turn about the same axis. Producers MUST then set `pano:roll` to 0 and express the
  whole turn in `pano:yaw`.

The angles are precisely defined in [Orientation](#orientation).

## Orientation

### The reference direction

A panorama looks in every direction at once. Saying that it faces north means nothing until it is clear which part of the image
faces north. Like GPano, this extension uses the viewing ray through the centre point of the image, called the
**reference direction**. See [Equirectangular images](#equirectangular-images) for images with an even number of pixels.

`pano:yaw`, `pano:pitch` and `pano:roll` are the heading, elevation and roll of that ray.

### The camera frame

Directions in the image are expressed in the panorama's camera frame. This is a right-handed Cartesian frame with its origin at the
projection centre:

- `x` points right of the reference direction;
- `y` points to the top of the image;
- `z` points backward, so the reference direction is `-z`.

This is the usual photogrammetric camera frame, and also the camera frame of OpenGL-style 3D engines. It is the
"image record coordinate system" that `pers:rotation_matrix` rotates into. The `pers:` README does not define that coordinate system
for panoramas, so this extension does.

### The world frame

The angles are measured in the local east-north-up frame of `pers:crs` at the projection centre. `pers:rotation_matrix` uses the
same frame, so every photogrammetric field of an Item shares it. The three axes are:

- `E` points towards increasing easting, or increasing longitude for a geographic CRS;
- `N` points towards increasing northing (*grid north*), or increasing latitude (*true north*) for a geographic CRS;
- `U` points up along the local vertical. The ellipsoid normal and the plumb line differ by seconds of arc, which is below what these
  angles resolve.

The axis order is always east, north, up, whatever axis order the CRS definition declares. For example, EPSG:4326 lists latitude
first. For the conformal projections used in surveying, such as Transverse Mercator, oblique stereographic and Lambert Conformal
Conic, grid east and grid north are perpendicular on the ground.

When `pers:crs` is absent it defaults to EPSG:4326, as in `pers:`, and north is true north. GPano uses the same frame, so its values
carry over unchanged. See [GPano](#gpano).

#### Grid north

When `pers:crs` is a projected CRS, north is **grid north**: the direction of the CRS's northing axis. Grid north is not true north.
The two differ by the meridian convergence `γ`:

```text
true heading = pano:yaw + γ
```

Here `γ` is the angle from true north clockwise to grid north at the projection centre. It reaches about 1.5° at the edges of the
Dutch RD grid (EPSG:28992), and 2 to 3° at the edge of a UTM zone. PROJ reports it as the meridian convergence, for example
`pyproj.Proj(crs).get_factors(lon, lat).meridian_convergence`.

This has two consequences:

- A client that draws the panorama on a WGS 84 or Web Mercator map and treats `pano:yaw` as a true-north heading is off by `γ`.
  That is usually acceptable for display. For measurement, work in `pers:crs`.
- A producer that re-expresses `pers:perspective_center` in another CRS MUST also re-derive the angles and `pers:rotation_matrix`
  for the new frame.

Using the frame of `pers:crs` keeps the angles, `pers:rotation_matrix` and `pers:perspective_center` in one frame. It is exact in the
CRS the panorama was surveyed in, and producers need no geodetic transformation to publish the angles.

### The rotation

Yaw, pitch and roll are Tait-Bryan angles, applied in that order:

1. yaw turns the camera clockwise about `U`;
2. pitch tilts it up about its own right-hand axis;
3. roll turns it about its own reference direction, right-hand edge down.

The rotation that takes camera coordinates to world coordinates is:

```text
R = Rz(-yaw) · Rx(pitch) · Ry(roll) · Rx(90°)
```

`Rx`, `Ry` and `Rz` are the standard right-handed rotations about `E`, `N` and `U`, for example
`Rx(a) = [[1, 0, 0], [0, cos a, -sin a], [0, sin a, cos a]]`. The fixed `Rx(90°)` places the camera frame so that at `(0, 0, 0)`
the reference direction `-z` points north and `y` points up.

`pers:rotation_matrix` rotates the world into the camera, so it is the transpose `M = Rᵀ`. Its rows are the image's right, up and
backward directions expressed in `E`, `N` and `U`. Writing `cy = cos(yaw)`, `sy = sin(yaw)`, `cp = cos(pitch)`, `sp = sin(pitch)`,
`cr = cos(roll)` and `sr = sin(roll)`:

```text
               E                     N                     U
right  [  cr·cy + sr·sp·sy    -cr·sy + sr·sp·cy    -sr·cp  ]
up     [  sr·cy - cr·sp·sy    -sr·sy - cr·sp·cy     cr·cp  ]
back   [ -sy·cp               -cy·cp               -sp     ]
```

The reference direction is therefore `(sin(yaw)·cos(pitch), cos(yaw)·cos(pitch), sin(pitch))` in `E`, `N`, `U`.

When both are present, `pano:yaw`, `pano:pitch`, `pano:roll` and `pers:rotation_matrix` MUST describe the same rotation, within
rounding. To recover the angles from `M = [m11, m12, m13, m21, m22, m23, m31, m32, m33]`:

```text
pitch = asin(-m33)
yaw   = atan2(-m31, -m32), wrapped to [0, 360)
roll  = atan2(-m13, m23)

and when cos(pitch) = 0:
roll  = 0
yaw   = atan2(-m12, m11), wrapped to [0, 360)
```

The `pers:` schema also accepts `pers:omega`, `pers:phi` and `pers:kappa`, but `pers:` does not define their convention, so this
extension cannot relate them to the angles. Prefer `pers:rotation_matrix`, which `pers:` itself recommends.

#### Quaternion

The same rotation `R` as a unit quaternion, in the Hamilton convention with the scalar first, `(w, x, y, z)`:

```text
q = qz(-yaw) ⊗ qx(pitch) ⊗ qy(roll) ⊗ qx(90°)

where qx(a) = (cos(a/2), sin(a/2), 0, 0)
      qy(a) = (cos(a/2), 0, sin(a/2), 0)
      qz(a) = (cos(a/2), 0, 0, sin(a/2))
```

`q` is an active rotation. It turns the `E`, `N`, `U` axes onto the camera's `x`, `y`, `z` axes, and it rotates a vector from camera
coordinates into world coordinates: `v_world = q ⊗ v_camera ⊗ q*`. An OpenGL-style camera placed in an east-north-up world (looking
along its `-z` axis, `y` up) takes `q` as its orientation. `pers:rotation_matrix` corresponds to the conjugate `q*`. `q` and `-q`
describe the same rotation; the test vectors below use `w ≥ 0`.

#### Reference implementation

In Python with SciPy 1.14 or later:

```python
from scipy.spatial.transform import Rotation


def camera_to_world(yaw, pitch, roll):
    """R: camera (x right, y up, z back) to world (east, north, up)."""
    return Rotation.from_euler("ZXY", [-yaw, pitch, roll], degrees=True) * Rotation.from_euler("x", 90, degrees=True)


r = camera_to_world(37.5, -12.25, 4.75)
rotation_matrix = r.as_matrix().T.flatten().tolist()  # pers:rotation_matrix
quaternion = r.as_quat(scalar_first=True).tolist()  # (w, x, y, z)
```

The uppercase `"ZXY"` selects intrinsic rotations, which gives the composition above.

#### Test vectors

```json
[
  { "yaw": 0, "pitch": 0, "roll": 0,
    "rotation_matrix": [1, 0, 0, 0, 0, 1, 0, -1, 0],
    "quaternion": [0.707106781, 0.707106781, 0, 0] },
  { "yaw": 90, "pitch": 0, "roll": 0,
    "rotation_matrix": [0, -1, 0, 0, 0, 1, -1, 0, 0],
    "quaternion": [0.5, 0.5, -0.5, -0.5] },
  { "yaw": 0, "pitch": 30, "roll": 0,
    "rotation_matrix": [1, 0, 0, 0, -0.5, 0.866025404, 0, -0.866025404, -0.5],
    "quaternion": [0.5, 0.866025404, 0, 0] },
  { "yaw": 0, "pitch": 0, "roll": 30,
    "rotation_matrix": [0.866025404, 0, -0.5, 0.5, 0, 0.866025404, 0, -1, 0],
    "quaternion": [0.683012702, 0.683012702, 0.183012702, -0.183012702] },
  { "yaw": 37.5, "pitch": -12.25, "roll": 4.75,
    "rotation_matrix": [0.7799326, -0.620609899, -0.080922756, 0.194418132, 0.117343287, 0.973874809,
                        -0.594900605, -0.775289563, 0.212177672],
    "quaternion": [0.7261979, 0.602165185, -0.176941385, -0.280580552] }
]
```

## Equirectangular images

An equirectangular image of `W × H` pixels covers the whole sphere. Pixel `(i, j)` is column `i` and row `j`, both counted from 0 at
the top left. In continuous image coordinates `(u, v)`, the pixel is the square from `(i, j)` to `(i + 1, j + 1)`, and its centre is
at `(i + ½, j + ½)`. The point `(u, v)` looks in this direction:

```text
longitude λ = 360° · (u / W - ½)      positive to the right
latitude  φ = 180° · (½ - v / H)      positive up

direction = (cos φ · sin λ,  sin φ,  -cos φ · cos λ)      in the camera frame
```

It follows that:

- **The reference direction is the image centre `(W/2, H/2)`.** When `W` and `H` are even, as they almost always are, this is the
  corner shared by the four middle pixels, not the centre of any pixel. Columns `W/2 - 1` and `W/2` lie half a pixel either side
  of it.
- **Column 0 starts at `yaw - 180°`.** The left edge of column 0 and the right edge of column `W - 1` both look straight back,
  at `λ = ±180°`. In an upright panorama that is azimuth `yaw - 180°`, and the centre of column `i` looks at azimuth
  `yaw - 180° + (i + ½) · 360° / W`.
- **The image is not mirrored.** Moving right in the image turns clockwise, as seen from above.
- **The top edge is the camera's zenith (`+y`) and the bottom edge is its nadir.**

Pixels are square when `W = 2H`.

## Assets

The fields in Item Properties describe the panorama that the Item represents. As usual in STAC, they apply to the Item's assets,
and an asset can override them. For example, a levelled copy of the panorama has `pano:pitch: 0` and `pano:roll: 0`.

Thumbnails are the exception. They are often a flat crop of the panorama rather than a small panorama. Panoramax thumbnails, for
example, are 500 × 300 perspective views. So an asset with the `thumbnail` role is not a panorama unless it carries
`pano:projection_type` itself.

The fields describe assets that are images of the panorama: at any resolution, whole or tiled, and in any band, including depth.
They do not apply to other assets, such as metadata files or point clouds.

## Relation to other extensions

### Perspective Imagery

- `pers:crs` and `pers:vertical_crs` define the [world frame](#the-world-frame).
- `pers:perspective_center` is the projection centre of the panorama, height included.
- `pers:rotation_matrix` MUST agree with the angles. See [The rotation](#the-rotation).
- `pers:interior_orientation.field_of_view: 360` is superseded by `pano:projection_type`. Producers MAY keep it for older clients.
- A snapshot is a perspective view cut from a panorama. Snapshots are frame images, so describe them with `pers:`, not with this
  extension.

### View

`view:azimuth` is a true-north bearing. Producers SHOULD set it for clients that do not know this extension, to the true heading
of the reference direction. That is `pano:yaw` when north is true north, and `pano:yaw + γ` when `pers:crs` is projected
(see [Grid north](#grid-north)). `view:off_nadir` has no meaning for a panorama and SHOULD NOT be set.

### GPano

This table maps the [GPano XMP fields](https://developers.google.com/streetview/spherical-metadata) to this extension:

| GPano                                | Panorama Extension                                                         |
| ------------------------------------ | -------------------------------------------------------------------------- |
| `ProjectionType` = `equirectangular` | `pano:projection_type` = `equirectangular`                                 |
| `PoseHeadingDegrees`                 | `pano:yaw`                                                                 |
| `PosePitchDegrees`                   | `pano:pitch`                                                               |
| `PoseRollDegrees`                    | `pano:roll`                                                                |
| `InitialViewHeadingDegrees`          | Not mapped. It is where a viewer starts, not how the image is oriented.    |
| `CroppedArea…`, `FullPano…`          | Not in this version. See [Future work](#future-work).                      |

GPano measures heading clockwise from true north. It defines roll so that the horizon rotates counterclockwise in the image as roll
increases, which is the same as the right-hand edge tilting down. Its ranges are the ones used here. So GPano angles are this
extension's angles in the default frame (no `pers:crs`, or EPSG:4326), and they copy across unchanged.

EXIF `GPSImgDirection` on a 360° camera often records the direction of travel, or the direction of one lens, rather than the image
centre. It can disagree with `PoseHeadingDegrees` by any amount. For example, the image in the
[Panoramax example](examples/item-panoramax.json) has `PoseHeadingDegrees` 198.6° and `GPSImgDirection` 359.9°. Use
`GPSImgDirection` for `pano:yaw` only when it is known to refer to the image centre.

## Best practices

- Use a `Point` at the projection centre as the Item geometry, and put the exact position, with height, in
  `pers:perspective_center`.
- Surveyed rigs measure pitch and roll. Publish them: they are what a viewer needs to show the sphere level.
- Expose `pano:projection_type` as a queryable in STAC APIs, so that clients can find panoramas with
  `pano:projection_type = 'equirectangular'`.
- In Collections, summarise `pano:projection_type` and the ranges of the angles.

## Future work

These are candidates for later versions. Issues and pull requests are welcome.

- **Cropped panoramas** (a v1.1.0 candidate): panoramas that cover less than the full sphere, as described by GPano's
  `CroppedArea…` and `FullPano…` fields. This was requested in
  [perspective-imagery#10](https://github.com/stac-extensions/perspective-imagery/issues/10). Contributions from Panoramax, who raised
  it, are especially welcome.
- **Meridian convergence**: a field that carries `γ`, so clients can get true north without PROJ.
- **A CRS-independent position**: an ellipsoidal projection centre next to `pers:perspective_center`.
- **More projections**, such as cylindrical and fisheye.

## Contributing

All contributions are subject to the
[STAC Specification Code of Conduct](https://github.com/radiantearth/stac-spec/blob/master/CODE_OF_CONDUCT.md).
For contributions, please follow the
[STAC specification contributing guide](https://github.com/radiantearth/stac-spec/blob/master/CONTRIBUTING.md). Instructions
for running tests are copied here for convenience.

### Running tests

The same checks that run on pull requests are part of the repository and can be run locally to verify that changes are valid.
To run tests locally, you'll need `npm`, which is a standard part of any [node.js installation](https://nodejs.org/en/download/).

First, install everything with npm once. Navigate to the root of this repository and on your command line run:

```bash
npm install
```

Then, to check markdown formatting and test the examples against the JSON schema, run:

```bash
npm test
```

This prints the same messages that you see online, so you can fix your markdown or examples.

If the tests reveal formatting problems with the examples, you can fix them with:

```bash
npm run format-examples
```
