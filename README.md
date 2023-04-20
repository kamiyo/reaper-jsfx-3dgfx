# 3D Graphics Utility Functions for Reaper JSFX

`v1.0`

A collection of functions to help you make 3d graphics in a Reaper Effects Plugin. Functions included for creating vectors/points and matrices, and a model-view-projection pipeline (inspired from OpenGL). Unfortunately, there is no depth-testing, so painter's algorithm should be used.

![Demo gif](3dgfx-320.gif)

This library requires [JackUtilities dynamic allocator](https://github.com/jack461/JackUtilities/blob/main/Utilities/mSL_Dyn_Memory.jsfx-inc), which also requires the [static allocator](https://github.com/jack461/JackUtilities/blob/main/Utilities/mSL_StM_Memory.jsfx-inc).

## Notes on memory usage

All code and checks assume the matrices are square and vectors are column. Most functions do not mutate inputs, but rather write outputs to a pointer. Geometric functions are in homogenous coordinates. The pointer can be created by calling `ptr = mSL_Dyn_Alloc(16, 'mats', mSL_StM_FlClear)`.

For allocator usage, please see:<br>
  * https://github.com/jack461/JackUtilities/blob/main/Utilities/mSL_Dyn_Memory.jsfx-inc<br>
  * https://github.com/jack461/JackUtilities/blob/main/Note-010-Dynamic-Memory-Allocation/JJ-Test-JSFX-Dyn-1.jsfx

## Basic Setup

Copy the GFX file into your user effects folder.

(Also be sure to copy JackUtilities folder into your effects, both the dynamic and static memory.)

Then, in your effects plugin file, something like this:
```
desc:MIDI visualization example
import JackUtilities/mSL_StM_Memory.jsfx-inc
import JackUtilities/mSL_Dyn_Memory.jsfx-inc
import GFX/gfx_functions.jsfx-inc

// Set input/output pins to none for MIDI-only
in_pin:none
out_pin:none

@init
mSL_StM_GlobFlgs=mSL_StM_FlFill+mSL_StM_FlSigErr+mSL_StM_FlCheck;
mSL_StM_Init(1024);
static_table = mSL_StM_Alloc(128, 'data', mSL_StM_FlClear);
mem = mSL_StM_BlockStart('XXXX');
dyn = mSL_Dyn_Init(mem-2, mem[-1]+4);

// Init the library. All functions are prefixed with gfxu_
gfxu_init();

// Create vectors for camera setup
eye = mSL_Dyn_Alloc(4, 'vecs', mSL_StM_FlClear);
at = mSL_Dyn_Alloc(4, 'vecs', mSL_StM_FlClear);
up = mSL_Dyn_Alloc(4, 'vecs', mSL_StM_FlClear);
gfxu_point(256, 170, 125, eye);
gfxu_point(176, 32, -64, at);
gfxu_vec(0, 1, 0, up);

/* Use the look_at function to create an inverse camera transformation matrix and store it in the gfxu_view matrix */
gfxu_look_at(eye, at, up);

/* Allocate some memory */
line_start = mSL_Dyn_Alloc(4, 'vecs', mSL_StM_FlClear);
line_end = mSL_Dyn_Alloc(4, 'vecs', mSL_StM_FlClear);

@gfx 720 360

view_w = gfx_w;
view_h = gfx_h;
aspect = view_w / view_h;
/* Transforms ([-1, 1], [-1, 1]) coords to actual screen pixels. This is applied last. */
gfxu_viewport(0, gfx_h, view_w, -view_h);

/* Projection matrix */
gfxu_perspective($pi / 4, aspect, 1, 100);

/* Precompute the M*V*P transformation matrix to take 3D -> 2D */
gfxu_compute_mvp();

/* Now you're ready to draw. Let's draw the x, y, and z axes, with a length of 100 */
gfx_set(255, 255, 255, 1.0);
gfxu_point(0, 0, 0, line_start);

// x
gfxu_point(100, 0, 0, line_end);
gfxu_draw_line(line_start, line_end);

// y
gfxu_point(0, 100, 0, line_end);
gfxu_draw_line(line_start, line_end);

// z
gfxu_point(0, 0, 100, line_end);
gfxu_draw_line(line_start, line_end);

```

Check out the `MIDI/midi_visualizer` file for a more involved example.

# API

There are quite a few "internal" utility functions, and you can probe around in the source to find those. The most important "public" functions will be listed here:
<br>
<br>

## Vectors and Matrices
<br>

### `gfxu_print_mat(s, mat)`

Prints a matrix or vector `mat` into string `s`.

>`s`: a string, like `#matrix`.<br>
`mat`: matrix or vector created with `mSL_Dyn_Alloc()`.

<br>

### `gfxu_dup_vec(v, res)`

Duplicates a matrix or vector `v` into `res`.

>`v, res`: matrices or vectors created with `mSL_Dyn_Alloc()`.

<br>

### `gfxu_vec(x, y, z, vec)`

Makes a 3d vector in 'vec' pointer. This sets `w`, or `vec[3]` to 0.

>`x, y, z`: numbers.<br>
`vec`: vector created with `mSL_Dyn_Alloc()`.

<br>

### `gfxu_point(x, y, z, vec)`

Makes a 3d point in 'vec' pointer. This sets `w`, or `vec[3]` to 1.

>`x, y, z`: numbers.<br>
`vec`: vector created with `mSL_Dyn_Alloc()`.

<br>

### `gfxu_make_i(mat)`

Stores an identity matrix into `mat`.

>`mat`: matrix created with `mSL_Dyn_Alloc()`.

<br>

### `gfxu_mat_vec_mult(mat, vec, res)`

Multiplies a matrix `mat` with a vector `vec` and stores it in `res`. This is also how you transform vectors. Error flag is set if sizes are not compatible.

>`mat`: matrix created with `mSL_Dyn_Alloc()`.<br>
`vec`: vector created with `mSL_Dyn_Alloc()`.<br>
`res`: vector created with `mSL_Dyn_Alloc()`.

<br>

### `gfxu_mat_mult(mat, mat2, res)`

Multiplies a matrix `mat` with another one `mat2` and stores it in `res`. Error flag is set if sizes are not compatible.

>`mat`: matrix created with `mSL_Dyn_Alloc()`.<br>
`mat2`: matrix created with `mSL_Dyn_Alloc()`.<br>
`res`: matrix created with `mSL_Dyn_Alloc()`.

<br>

### `gfxu_scalar_mat_mult(scalar, mat, res)`

Multiplies a matrix `mat` by a scalar `scalar` and stores it in `res`. Error flag is set if sizes are not compatible.

>`scalar`: a number.<br>
`mat`: matrix created with `mSL_Dyn_Alloc()`.<br>
`res`: matrix created with `mSL_Dyn_Alloc()`.

<br>

### `gfxu_add_vec(v1, v2, res)`

Adds two vectors (or matrices) `v1` and `v2` and stores it in `res`. Error flag is set if sizes are not compatible.

>`v1, v2, res`: vectors/matrices created with `mSL_Dyn_Alloc()`.<br>

<br>

### `gfxu_sub_vec(v1, v2, res)`

Subtracts `v2` from `v1` and stores it in `res`. Error flag is set if sizes are not compatible.

>`v1, v2, res`: vectors/matrices created with `mSL_Dyn_Alloc()`.<br>

<br>

### `gfxu_dot(v1, v2)`

Returns the dot product of `v1` and `v2`. Error flag is set if sizes are not compatible.

>`v1, v2`: vectors created with `mSL_Dyn_Alloc()`.<br>

<br>

### `gfxu_norm(v)`

Returns the norm of vector `v`.

>`v`: vector created with `mSL_Dyn_Alloc()`.<br>

<br>

### `gfxu_normalize(v, res)`

Normalizes vector `v` and stores it in `res`. Error flag is set if sizes are not compatible.

>`v, res`: vectors created with `mSL_Dyn_Alloc()`.<br>

<br>

### `gfxu_cross(v1, v2, res)`

Computes the 3d cross product of `v1` and `v2` and stores it in `res`.

>`v1, v2, res`: vectors created with `mSL_Dyn_Alloc()`.<br>

<br>

### `gfxu_rot_mad_3d(axis, angle, mat)`

Returns the rotation matrix for a rotation of `angle` about `axis` and stores it in `mat`. Error flag is set if sizes are not compatible.

>`axis`: vector created with `mSL_Dyn_Alloc()`.<br>
`angle`: rotation angle in radians.<br>
`mat`: matrix created with `mSL_Dyn_Alloc()`.

<br>

## Graphics Pipeline API
<br>

### `gfxu_init()`

Call before using any `gfxu_` functions, so that temporary, global, and swap memory can be allocated.

<br>

### `gfxu_compute_mvp()`

Call before using any `gfxu_draw_` functions to pre-multiply `gfxu_proj * gfxu_view * gfxu_model` and store it in `gfxu_mvp`.

<br>

## Model Transforms

The following act on the `gfxu_model` matrix.

<br>

### `gfxu_scale(scalar)`

Pre-multiplies the `gfxu_model` matrix with a `kI` where `k` is the scalar, and `I` is the identity matrix. Stores back in `gfxu_model`.

>`scalar`: number.

<br>

### `gfxu_rotate(axis, angle)`

Pre-multiplies the `gfxu_model` matrix with a rotation matrix calculated with `gfxu_rot_mat_3d` and stores it back in `gfxu_model`.

>`axis`: vector created with `mSL_Dyn_Alloc()`.<br>
`angle`: rotation angle in radians.<br>

<br>

### `gfxu_translate(vec)`

Pre-multiplies the `gfxu_model` matrix with a translation matrix and stores it back in `gfxu_model`.

>`vec`: vector created with `mSL_Dyn_Alloc()`.

<br>

### `gfxu_transform(mat)`

Pre-multiplies the `gfxu_model` matrix with an arbitrary matrix and stores it back in `gfxu_model`.

>`mat`: matrix created with `mSL_Dyn_Alloc()`.

<br>

## View Transforms

All of the following act on the `gfxu_view` matrix.

<br>

### `gfxu_look_at(eye, at, up)`

See OpenGL's gluLookAt for explanation. Calculates projection matrix for camera at `eye` looking at `at`. `up` vector is supplied for calculating the camera's orthonormal basis.

>`eye, at, up`: vectors created with `mSL_Dyn_Alloc()`.

<br>

### `gfxu_arcball_rot(dx, dy)`

Utility function for moving the camera along an arcball surface.

>`dx, dy`: angles in radians.

<br>

### `gfxu_arcball_move_xz(dx, dz)`

Move the camera's `at` location along the xz-plane

>`dx, dz`: position change in object-space units.

<br>

### `gfxu_arcball_move_xy(dx, dy)`

Move the camera's `at` location along the xy-plane

>`dx, dy`: position change in object-space units.

<br>

### `gfxu_zoom(zoom)`

Move the camera towards or away from the `at` point by scaling `eye - at` by `zoom` amount. Only works with perspective projection. To "zoom" orthographic projection, manipulate the orthographic projection bounds.

>`zoom`: percentage of original `eye - at` length to scale.

<br>

## Projection Transforms

These functions overwrite the current `gfxu_proj` matrix.

### `gfxu_ortho(left, right, bottom, top, near, far)`

Calculates an orthographic projection from parameters specified.

>All params are numbers, specifying in object-space coordinates the bounds of the projection.

<br>

### `gfxu_perspective(fovy, aspect, zNear, zFar)`

Calculates a perspective projection transform.

>`fovy`: vertical field-of-view in radians.<br>
`aspect`: width/height of the frustrum.<br>
`zNear, zFar`: distance to the nearest and farthest points, respectively, of the frustrum.<br>

<br>

## Viewport Transforms

<br>

## gfxu_viewport(x, y, width, height)

Sets the matrix that will convert normalized device coordinates into viewport coordinates.

>`x, y`: The lower-left corner in screen pixels of the viewport.
`width, height`: Dimensions of the viewport.

<br>

## Drawing Functions

### `gfxu_draw_line(start, end)`

Draws a line from object-space positions `start` to `end`.

>`start, end`: vectors (points) created with `mSL_Dyn_Alloc()`.

<br>

### `gfxu_draw_triangle(p1, p2, p3)`

Draws a triangle using the three points.

>`p1, p2, p3`: vectors (points) created with `mSL_Dyn_Alloc()`.

<br>

### `gfxu_draw_circle(p, r)`

Draws a circle of `r` screen-units radius at object-space `p`.

>`p`: vector (point) created with `mSL_Dyn_Alloc()`.<br>
`r`: radius of circle in screen-units (pixels).

<br>

### `gfxu_draw_parallelogram(start, w_vec, h_vec)`

Draws a parallelogram specified by a starting point and two vectors.

>`start`: vector (point) created with `mSL_Dyn_Alloc()`.<br>
`w_vec, h_vec`: vectors representing the adjacent sides of the shape.

<br>

##  Error Codes

Error codes (bit set, 0-indexed):

0.   Matrix access out-of-bounds
1.   Matrix x vector input and/or output size mismatch
2.   Matrix transpose input/output size mismatch
3.   Matrix x matrix input and/or output sizes mismatch\
4.   Scalar multiply input/output size mismatch
5.   Vector addition input and/or output sizes mismatch
6.   Vector subtraction input and/or output sizes mismatch
7.   Dot product input sizes mismatch
8.   Normalize input/output size mismatch
9.   Vector/Matrix copy input/output size mismatch
10.  Comparison input/output size mismatch
11.  Rotation matrix output must be at least 3x3
