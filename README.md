# Panorama Extension Specification

- **Title:** Panorama
- **Identifier:** <https://raw.githubusercontent.com/360-geo/stac-panorama/main/json-schema/schema.json>
- **Field Name Prefix:** pano
- **Scope:** Item, Collection
- **Extension [Maturity Classification](https://github.com/radiantearth/stac-spec/tree/master/extensions/README.md#extension-maturity):** Proposal
- **Owner**: @georgeboot

This document explains the Panorama Extension to the [SpatioTemporal Asset Catalog](https://github.com/radiantearth/stac-spec) (STAC)
specification. It marks an Item as a 360° panorama and gives its orientation as yaw, pitch and roll, with the same meaning as the GPano
`PoseHeadingDegrees`, `PosePitchDegrees` and `PoseRollDegrees` fields. It complements
[Perspective Imagery](https://github.com/stac-extensions/perspective-imagery) (`pers:`), which describes frame cameras.

- Examples:
  - [Georizon Item](examples/item-georizon.json): a panorama in a projected CRS (RD New + NAP), with `pers:` fields
  - [Panoramax Item](examples/item-panoramax.json): a panorama with GPano angles, without `pers:crs`
  - [Collection example](examples/collection.json): summaries and item asset definitions
- [JSON Schema](json-schema/schema.json)
- [Changelog](./CHANGELOG.md)

## Fields

The fields in the table below can be used in these parts of STAC documents:

- [ ] Catalogs
- [ ] Collections
- [x] Item Properties (incl. Summaries in Collections)
- [x] Assets (for both Collections and Items, incl. Item Asset Definitions in Collections)
- [ ] Links

| Field Name           | Type   | Description |
| -------------------- | ------ | ----------- |
| pano:projection_type | string | **REQUIRED**. Projection of the panoramic image. One of: `equirectangular`. |
| pano:yaw             | number | Heading of the image centre in degrees, clockwise from north, `[0, 360)` |
| pano:pitch           | number | Elevation of the image centre above the horizontal in degrees, `[-90, 90]` |
| pano:roll            | number | Rotation of the image about its centre in degrees, `(-180, 180]`. Positive tilts the right-hand edge down. |

### Additional Field Information

#### pano:projection_type

`equirectangular` is an image of the full sphere, 360° wide and 180° high. This field replaces the practice of setting
`pers:interior_orientation.field_of_view` to 360. Assets with the `thumbnail` role are not assumed to be panoramic, since thumbnails
are often flat crops.

#### Orientation

The angles describe the viewing ray through the centre of the image, which is the point `(width / 2, height / 2)`. Column 0 starts
at `yaw - 180°`.

- **Origin:** at `(0, 0, 0)`, the image centre looks north at the horizon and the image is upright.
- **Order:** yaw turns clockwise about the vertical axis. Pitch then tilts the view up about the image's right-hand axis. Roll then
  turns the image about its viewing ray.
- **World axes:** `x` east, `y` north, `z` up.
- **Image axes:** `x` right, `y` up, `z` backward (the viewing ray is `-z`).

Together, the rotation from image to world is `Rz(-yaw) · Rx(pitch) · Ry(roll) · Rx(90°)`. `pers:rotation_matrix` is its
transpose, and MUST describe the same rotation when present.

The world axes are the axes of `pers:crs` when it is set. For a projected CRS, north is therefore grid north: the true heading is
`pano:yaw` plus the meridian convergence. Without `pers:crs`, north is true north, as in GPano.

## Contributing

All contributions are subject to the
[STAC Specification Code of Conduct](https://github.com/radiantearth/stac-spec/blob/master/CODE_OF_CONDUCT.md).
For contributions, please follow the
[STAC specification contributing guide](https://github.com/radiantearth/stac-spec/blob/master/CONTRIBUTING.md) Instructions
for running tests are copied here for convenience.

### Running tests

The same checks that run as checks on PRs are part of the repository and can be run locally to verify that changes are valid.
To run tests locally, you'll need `npm`, which is a standard part of any [node.js installation](https://nodejs.org/en/download/).

First you'll need to install everything with npm once. Just navigate to the root of this repository and on
your command line run:

```bash
npm install
```

Then to check markdown formatting and test the examples against the JSON schema, you can run:

```bash
npm test
```

This will spit out the same texts that you see online, and you can then go and fix your markdown or examples.

If the tests reveal formatting problems with the examples, you can fix them with:

```bash
npm run format-examples
```
