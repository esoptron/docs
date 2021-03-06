# Shaders
Libretro has created this Shader Specification in order to facilitate the implementation of cross-platform video shaders.

#### Available shader types:

* **Cg shaders**: The Cg shader spec used in RetroArch and other libretro frontends supports both single-pass Cg shaders as well as multi-pass shaders. It uses a custom Cg preset format (.cgp).
* **GLSL shaders**: GLSL support exists for platforms which do not support Cg shaders, which is the case for OpenGL ES, and EGL contexts including KMS mode in Linux.

#### Shader file format:

    # are treated as comments. Rest of the line is ignored.
    Format is: key = value. There can be as many spaces as you like in-between.
    Value can be wrapped inside "" for multiword strings. (foo = "hai u")
    #include includes a config file in-place.
    Path is relative to where config file was loaded unless an absolute path is chosen.
    Key/value pairs from an #include are read-only, and cannot be modified.

## Cg shader spec
Cg is a shader specification from nVidia. It has the advantage that shaders written in Cg are compatible with both OpenGL and Direct3D, as well as PlayStation3. They are also compatible with basic HLSL if some considerations are taken into account. They can even be automatically compiled into GLSL shaders, which makes Cg shaders a true “write once, run everywhere” shader format. We encourage new shaders targeting Libretro frontends to be written in this format.

### Example Cg shader

    void main_vertex
    (
       float4 position : POSITION,
       out float4 oPosition : POSITION,
       uniform float4x4 modelViewProj,
       float2 tex : TEXCOORD,
       out float2 oTex : TEXCOORD
    )
    {
       oPosition = mul(modelViewProj, position);
       oTex = tex;
    }
    
    float4 main_fragment (float2 tex : TEXCOORD, uniform sampler2D s0 : TEXUNIT0) : COLOR
    {
       return tex2D(s0, tex);
    }

### Cg shader presets

#### Example Cg preset
    shaders = 2
    shader0 = 4xBR-v3.9.cg
    scale_type0 = source
    scale0 = 4.0
    filter_linear0 = false
    shader1 = dummy.cg
    filter_linear1 = true

## GLSL shader spec
Like Cg shaders, GLSL shaders represents a single pass, and requires a preset file to describe how multiple shaders are combined. The extension is .glsl.

As GLSL shaders are normally placed in two different files (vertex, fragment), making it very impractical to select in a menu. This is worked around by using compiler defines in order to be equivalent to Cg shaders.

### Example GLSL shader
Note: GLSL shaders must be modern style, and using ruby prefix is discouraged.

    varying vec2 tex_coord;
    #if defined(VERTEX)
    attribute vec2 TexCoord;
    attribute vec2 VertexCoord;
    uniform mat4 MVPMatrix;
    void main()
    {
        gl_Position = MVPMatrix * vec4(VertexCoord, 0.0, 1.0);
        tex_coord = TexCoord;
    }
    #elif defined(FRAGMENT)
    uniform sampler2D Texture;
    void main()
    {
        gl_FragColor = texture2D(Texture, tex_coord);
    }
    #endif

### GLSL presets

Like Cg shaders, there is a preset format. Instead of .cgp extension, .glslp extension is used. The format is exactly the same, just replace .cg shaders with .glsl. To convert a .cgp preset, rename to .glslp and replace all references to .cg shaders with .glsl.

## Converting from Cg shaders

GLSL shaders are mostly considered a compatibility format. It is possible to compile Cg shaders into GLSL shaders automatically using our [cg2glsl script](https://github.com/libretro/RetroArch/blob/master/tools/cg2glsl.py). It can convert single shaders as well as batch conversion. Shader converstion relies on nVidia's cgc tool found in the `nvidia-cg-toolkit` package.

## Common Shaders Repository

The Libretro organization hosts a [repository](https://github.com/libretro/common-shaders) on Github that contains a compilation of shaders. Users can contribute their own shaders to this repository by doing a Pull Request.

## Detailed Cg Shader Specification

Entry points:
   Vertex: main_vertex
   Fragment: main_fragment

Texture unit:
   All shaders work on texture unit 0 (the default). 2D textures must be used.
   Power-of-two sized textures are recommended for optimal visual quality.
   The shaders must deal with the actual picture data not 
   filling out the entire texture.
   Incoming texture coordinates and uniforms provide this information.

   The texture coordinate origin is defined to be top-left oriented, i.e.
   a texture coordinate of (0, 0) will always refer to the top-left pixel of
   the visible frame. This is opposite of what most graphical APIs expect.
   The implementation must always ensure that this ordering is held for any
   texture that the shader has access to.

   Every texture bound for a shader must have black border mode set.
   I.e. sampling a texel outside the given texture coordinates must always return a pixel
   with RGBA values (0, 0, 0, 0).

Uniforms:
   Some parameters will need to be passed to all shaders, 
   both vertex and fragment program.
   A generic entry point for fragment shader will look like:

   float4 main_fragment (float2 tex : TEXCOORD0, 
      uniform input IN, uniform sampler2D s_p : TEXUNIT0) : COLOR
   {}

   The input is a struct looking like:
   struct input
   {
      float2 video_size;
      float2 texture_size;
      float2 output_size;
      float frame_count;
      float frame_direction;
   };

   TEXCOORD0: Texture coordinates for the current input frame will be passed in TEXCOORD0.
   (TEXCOORD is a valid alias for TEXCOORD0).

   COLOR0: Although legal, no data of interest is passed here.
   You cannot assume anything about data in this stream.

   IN.video_size: The size of the actual video data in the texture, 
   e.g for a SNES this will be generally 
   (256, 224) for normal resolution frames.

   IN.texture_size: This is the size of the texture itself. 
   Optimally power-of-two sized.

   IN.output_size: The size of the video output. 
   This is the size of the viewport shown on screen.

   IN.frame_count: A counter of the frame number. 
   This increases with 1 every frame. 
   This value is really an integer, 
   but needs to be float for CGs lack of integer uniforms.

   IN.frame_direction: A number telling which direction
   the frames are flowing. For regular playing, this value should be 1.0.
   While the game is rewinding, this value should be -1.0.

   modelViewProj: This uniform needs to be set in vertex shader. 
   It is a uniform for the current MVP transform.

Pre-filtering:
   Most of these shaders are intended to be used with a non-filtered input. 
   Nearest-neighbor filtering on the textures themselves are preferred.
   Some shaders, like scanline will most likely 
   prefer bilinear texture filtering.

## Cg meta-shader format

Rationale:
The .cg files themselves contain no metadata necessary to perform advanced
filtering. They also cannot process an effect in multiple passes, which
is necessary for some effects. The CgFX format does exist, but it would
need current shaders to be rewritten to a HLSL-esque format.
It also suffers a problem mentioned below.

Rather than putting everything into one file (XML shader format), this
format is config file based. This greatly helps testing shader combinations
as there is no need to rearrange code in one big file. Another plus with
this approach is that a large library of .cg files can be used to combine
many shaders without needing to redundantly copy code over. It also
helps testing as it is possible to unit-test every pass separately 
completely seamless.

Format:

The meta-shader format is based around the idea of a config file with the
format: key = value. Values with spaces need to be wrapped in quotes:
key = "value stuff". No .ini sections or similar are allowed. Meta-shaders
may include comments, prefixed by the "#" character, both on their own in
an otherwise empty line or at the end of a key = value pair.

The meta-format has four purposes:
-  Combine several standalone .cg shaders into a multipass shader.
-  Define scaling parameters for each pass. I.e., a HQ2x shader
   might want to output with a scale of exactly 2x.
-  Control filtering of textures. Many shaders will want nearest-neighbor
   filtering, and some will want linear.
-  Define external lookup textures. Shaders can access external textures
   found in .tga files.

Parameters:

-  shaders (int): This param defines how many .cg shaders will be loaded.
   This value must be at least one.
   The path to these shaders will be found as a string in parameters
   shader0, shader1, ... shaderN, and so on.
   The path is relative to the directory the meta-shader
   was loaded from.

-  filter_linearN (boolean): This parameter defines how the texture of the 
   result of pass N will be filtered.
   N = 0 (pass 0) is the raw input frame,
   N = 1 is result of the first pass, etc.
   (A boolean value here might be true/false/1/0).
   Should this value not be defined, the filtering option is
   implementation defined.

-  float_framebufferN (boolean): This parameters defines if shader N
   should render to a 32-bit floating point buffer.
   This only takes effect if shaderN is actually rendered to an FBO.
   This is useful for shaders which have to store FBO values outside [0, 1] range.

-  frame_count_modN (int): This positive parameter defines
   which modulo to apply to IN.frame_count.
   IN.frame_count will take the value frame_count % frame_count_modN.

-  scale_typeN (string): This can be set to one of these values:
   "source":   Output size of shader pass N is relative to the input size
               as found in IN.video_size. Value is float.
   "viewport": Output size of shader pass N is relative to the size of the
               window viewport. Value is float.
               This value can change over time if the user 
               resizes his/her window!
   "absolute": Output size is statically defined to a certain size.
               Useful for hi-res blenders or similiar.

   If no scale type is assumed, it is assumed that it is set to "source"
   with scaleN set to 1.0.

   It is possible to set scale_type_xN and scale_type_yN to specialize
   the scaling type in either direction. scale_typeN however
   overrides both of these.

   Exceptions:
   If no scale_type is set for the very last shader,
   it is assumed to output at the full resolution rather than assuming
   a scale of 1.0x, and bypasses any frame-buffer object rendering. 
   If there is only one shader, it is
   also considered to be the very last shader. If any scale option
   is defined, it has to go through a frame-buffer object, and
   subsequently rendered to screen. The filtering option used when stretching
   is implementation defined. It is encouraged to not have any
   scaling parameters in last pass if you care about the filtering
   option here.

   In first pass, should no scaling factor be defined, the implementation
   is free to choose a fitting scale. This means, that for a single pass
   shader, it is allowed for the implementation to set a scale, 
   render to FBO, and stretch. (Rule above).

-  scaleN, scale_xN, scale_yN (float/int):
   These values control the scaling params from scale_typeN.
   The values may be either floating or int depending on the type.
   scaleN controls both scaling type in horizontal and vertical directions.

   If scaleN is defined, scale_xN and scale_yN have no effect.
   scale_xN and scale_yN controls scaling properties for the directions
   separately. Should only one of these be defined, the other direction
   will assume a "source" scale with value 1.0, i.e. no change in resolution.

   Should scale_type_xN and scale_type_yN be set to different values,
   the use of scaleN is undefined (i.e. if X-type is absolute (takes int),
   and Y-type is source (takes float).)

-  textures (multiple strings):
   The textures param defines one or more lookup textures IDs.
   Several IDs are delimited with ';'. I.e. textures = "foo;bar"
   These IDs serves as the names for a Cg sampler uniform. I.e.
   uniform sampler2D foo;
   uniform sampler2D bar;

   The path of the textures can be found in the IDs, i.e.
   foo = image0.tga
   bar = image1.tga
   The paths of these textures are relative to the directory
   the meta-shader was loaded from.

   It is also possible to control the filtering options of the
   lookup texture as a boolean option in ID_linear = true/false.
   I.e. foo_linear = false, will force nearest neighbor filtering
   for texture "foo". If this param is not set, it is assumed to be
   linearily filtered.

   The textures will be loaded "as-is", 
   and coordinates (0, 0), (0, 1), (1, 0), (1, 1) will correspond
   to the corners of the texture. Since the texture coordinates
   of the texture in TEXUNIT0 might not be as convenient,
   the texture coordinates for all lookup textures will be found
   in TEXCOORD1. (Note: You cannot assume which texture unit the
   lookup textures will be bound to however!)

   The implementation only guarantees to be able to
   load plain top-left non-RLE .tga files. 
   It may provide possibilities to load i.e. .png and other popular formats.

Multi-pass uniforms:

   During multi-pass rendering, some additional uniforms are available.

   With multi-pass rendering, it is possible to utilize the resulting output
   for every pass that came before it, including the unfiltered input.
   This allows for an additive approach to shading rather than
   serial style.

   The unfiltered input can be found in the ORIG struct:

   uniform sampler2D ORIG.texture: Texture handle. 
   Must not be set to a predefined texture unit.
   uniform float2 ORIG.video_size: The video size of original frame.
   uniform float2 ORIG.texture_size: The texture size of original frame.
   in float2 ORIG.tex_coord: An attribute input holding the texture
                             coordinates of original frame.

   PASS%u: This struct holds the same data as the ORIG struct,
   although the result of passes {1, 2, 3 ...}, i.e.
   PASS1.texture holds the result of the first shader pass.
   If rendering pass N, passes {1, ..., N-2} are available.
   (N-1 being input in the regular IN structure).

   PREV: 
   This struct holds the same data as the ORIG struct,
   and corresponds to the raw input image from the previous frame.
   Useful for motion blur.

   PREV1..6:
   Similar struct as PREV, but holds the data for passes further back in time.
   PREV1 is the frame before PREV, PREV2 the frame before that again, and so on.
   This allows up to 8-tap motion blur.

For backend implementers:

## Rendering the shader chain

With all these options, the rendering pipeline can become somewhat complex.
The meta-shader format greatly utilizes the possibility of offscreen
rendering to achieve its effects. 
In OpenGL usually this is referred to as frame-buffer objects,
and in HLSL as render targets (?). This feature will be referred to as
FBO from here. FBO texture is assumed to be a texture bound to the FBO.

As long as the visual result is approximately identical,
the implementation does not have
to employ FBO.

With multiple passes our chain looks like this conceptually:

|Source image| ---> |Shader 0| ---> |FBO 0| ---> |Shader 1| ---> 
|FBO 1| ---> |Shader 2| ---> (Back buffer)

In the case that Shader 2 has set some scaling params, we need to first render
to an FBO before stretching it to the back buffer.

|Source image| ---> ... |Shader 2| ---> |FBO 2| ---> (Back buffer)

Scaling parameters determine the sizes of the FBOs. For visual fidelity it
is recommended that power-of-two sized textures are bound to them. This
is due to floating point inaccuracies that become far more apparent when not
using power-of-two textures. If the absolute maximum size of the source image
is known, then it is possible to preallocate the FBOs.

Do note that the size of FBOn is determined by dimensions of FBOn-1 
when "source" scale is used, _not_ the source image size! 
Of course, FBO0 would use source image size, as there is no FBO-1 ;)

I.e., with SNES there is a maximum width of 512 and height of 478.
If a source relative scale of 3.0x is desired for first pass, it is thus safe to
allocate a FBO with size of 2048x2048. However, most frames will just use a
tiny fraction of this texture.

With "viewport" scale it might be necessary to reallocate the FBO in run-time
if the user resizes the window.


## Compatibility of shader types with context drivers

In RetroArch, the following table specifies which shader types work with what video contexts:

CONTEXT DRIVER         |    GLSL    |    CG   |    HLSL        |     SLANG
-----------------------|------------|---------|----------------|--------------
Android                |    Y       |    N    |    N           |  Y (Possible)
CGL                    |    Y       |    N    |    N           |  N
D3D                    |    N       |    Y    |    N (Possible)| N
DRM                    |    Y       |    N    |    N           | N
Emscripten             |    Y       |    N    |    N           | N
GDI                    |    N       |    N    |    N           | N
KHR                    |    N       |    N    |    N           | Y
Mali                   |    Y       |    N    |    N           | N
Opendingux             |    Y       |    N    |    N           | N
OSMesa                 |    Y       |    N    |    N           | N
PS3                    |    N       |    Y    |    N           | N
QNX                    |    Y       |    N    |    N           | N
SDL                    |    N       |    N    |    N           | N
VC                     |    Y       |    N    |    N           | N
Vivante                |    Y       |    N    |    N           | N
Wayland                |    Y       |    N    |    N           | Y
WGL                    |    Y       |    N    |    N           | Y
X                      |    Y       |    Y    |    N           | Y
XEGL                   |    Y       |    N    |    N           | N

Attempting to load unsupported shader types may result in segmentation faults because the context drivers currently do not have the behavior to declare which types of shaders it supports
~~~~
