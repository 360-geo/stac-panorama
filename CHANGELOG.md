# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

First version, to be released as v1.0.0.

### Added

- `pano:projection_type` (`equirectangular`), required in Item Properties, superseding
  `pers:interior_orientation.field_of_view = 360` as the panorama marker
- `pano:yaw`, `pano:pitch` and `pano:roll`: the orientation of the viewing ray through the image centre, compatible with GPano
- The world frame: the east-north-up axes of `pers:crs`, true north when `pers:crs` is absent, and grid north for a projected CRS
- The camera frame, the rotation matrix, the quaternion and the agreement rule with `pers:rotation_matrix`, with test vectors
- The pixel mapping of equirectangular images
- Asset inheritance, with thumbnails excluded unless they carry `pano:projection_type`
- Examples: a Georizon Item in RD New + NAP, a Panoramax Item and a Collection

[Unreleased]: <https://github.com/360-geo/stac-panorama/commits/main>
