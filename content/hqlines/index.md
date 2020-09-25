+++
title = "Is it easy to draw a line"
date = 2020-09-14
+++
<!-- ![lines_square](lines_square.jpg) -->

[Demo](https://pum-purum-pum-pum.github.io/lines/) Note for some reason on web
it's not that good as on desktop. I also had to turn off browsers AA manually.

[Source code](https://github.com/pum-purum-pum-pum/Lines)

### Wait what?

If you just draw rectangle in OpenGL you'll
notice that the edges look like a pixel ladder.
You may try to fix that with MSAA but it's still poor quality.
Also one may want to draw the line not as a rectangle but as a rounded rectangle.

![line](line.gif)

[picture reference](https://www.displaydaily.com/?view=article&id=102:antialiasing&catid=118:word-of-the-week)

### Normal + width

To draw just rectangle lines one can use normal to a segment as a vertex attribute.
This approach is well described [here](https://blog.mapbox.com/drawing-antialiased-lines-with-opengl-8766f34192dc)

### SDF

The other way of drawing high quality segments is using Signed Distance Field (SDF).
We can create smooth edges by calculating the distance to it.
Also, with SDF we can create a nice rounded segment's ends.

We also want to draw polygonal chains.
Unfortunately, we can't just draw these segments on the top of each other,
because of transparent edges that will overlap and create ugly blending effect.
Or if the line itself is transparent we will just see overlaps.
It can be solved with rendering to texture for example.

We will only consider opaque lines.

[Here](https://www.shadertoy.com/view/Wlfyzl)
is the shader we will use for the segment's SDF.

```Rust
float line_segment(in vec2 p, in vec2 a, in vec2 b) {
    vec2 ba = b - a;
    vec2 pa = p - a;
    float h = clamp(dot(pa, ba) / dot(ba, ba), 0., 1.);
    return length(pa - h * ba);
}
...
// later used like
lowp float distance = line_segment(pp, a, b) - thickness;
```

As you can see- in fragment shader we would need segment's ends and thickness.
That's why we need to pass these parameters as vertex attributes in the vertex
shader and then into fragment shader.
There are two ways that come to my head to do this:

1) Create a buffer with two triangles for each line.
Just pass SDF parameters in each Vertex.
2) Only create one buffer for a line with two triangles and pass
these parameters per line via instancing.

Our line struct will look like this. (segment type is actually discrete value)

```Rust
#[derive(Debug, Clone, Copy)]
#[repr(C)]
pub struct Line {
    pub segment_type: f32,
    pub position: Vec2,
    pub thickness: f32,
    pub dir: Vec2,
    pub color: Vec3,
}

impl Line {
    pub fn new(
        segment_type: SegmentType,
        from: Vec2,
        to: Vec2,
        thickness: f32,
        color: Vec3,
    ) -> Self {
        let dir = to - from;
        Line {
            segment_type: segment_type as u8 as f32, // dirty hack to make OpenGL happy
            position: (from + to) / 2.,
            thickness,
            dir,
            color,
        }
    }
}
```

### Shaders

I'm not sure if the second is faster (since the instance is only two triangles --
it's a small overhead).
But it's convenient and fast enough
to draw million of antialiased high-quality lines.

Here is the vertex shader code (sorry for the old glsl version):

```GLSL
#version 100
precision lowp float;
attribute vec2 pos;
attribute float segment_type;
attribute vec2 inst_pos;
attribute float thickness;
attribute vec2 dir;
attribute vec3 color0;

varying vec2 local_position;
varying vec2 projected_position;
varying vec2 ip;
varying float th;
varying vec4 color;
// segment type. Have to pass as float, but it is just enum
varying float st;
varying vec2 dr;

uniform mat4 mvp;
void main() {
    vec2 n = vec2(-dir.y, dir.x) / length(dir);
    vec2 apos = pos.y * dir + pos.x * n * thickness;
    vec4 new_pos = vec4(apos + inst_pos, 0.0, 1.0);
    vec4 res_pos = mvp * new_pos;
    gl_Position = res_pos;

    st = segment_type;
    local_position = pos;
    projected_position = vec2(new_pos.x, new_pos.y);
    ip = inst_pos;
    dr = dir;
    th = thickness;
    color = vec4(color0, 0.5);
}
```

We just change the shape of our line according to parameters:

- dir: segment direction
- thickness: thickness of the line

and then pass required variables into the fragment shader

Here is the fragment shader:

```glsl
#version 100
precision lowp float;
varying vec2 local_position;
varying vec2 projected_position;
varying vec2 ip;
varying float th;
varying vec4 color;
varying float st;
varying vec2 dr;

uniform mat4 mvp;
const lowp float aaborder = 0.00445;

float line_segment(in vec2 p, in vec2 a, in vec2 b) {
    vec2 ba = b - a;
    vec2 pa = p - a;
    float h = clamp(dot(pa, ba) / dot(ba, ba), 0., 1.);
    return length(pa - h * ba);
}

void main() {
    vec2 a = ip - dr  / 2.;
    vec2 b = ip + dr / 2.;
    float d = line_segment(projected_position, a, b) - th;
    float scaled_border = aaborder / mvp[1][1];
    float edge1 = -scaled_border;
    float edge2 = 0.;

    if (d < 0.) {
        float smooth = 1.;
        if (abs(st - 1.) < 0.01 && local_position.y < -0.5) { // in SDF space
            discard;
        } else if (abs(st - 2.) < 0.01 && local_position.y > 0.5) {
            discard;
        }
        if (d > edge1) {
            smooth = 1. - smoothstep(edge1, edge2, d) + st - st;
        }
        vec4 color = color;
        color.a = smooth;
        gl_FragColor = color;
    } else {
        gl_FragColor = vec4(color.xyz, 0.0);
    }
}
```

First, we do the same operation (create vertex position in world space) as in
vertex shader, but without projection with MVP matrix.
Then we calculate the distance to the borders with our line SDF function.
After that, we want to create a nice antialiased edge.
We pass the distance smoothstep function between `edge1` and `edge2` and then
pass the result value to the alpha channel of the result line's color.

In order to draw monochromatic polygonal chains with multiple segments we can't
just draw segments with corners on the top of each other.
Because if we do so then our smooth edges will overlap and blend with each other.
Maybe one can solve it with rendering to a texture or some nonstandard blending.
But we will just draw only one segment edge as shown in this picture
![overlap](overlap.jpg)
(note that each line has a different color only for demonstration.
We can draw a red line on the top of the yellow. Or clip red line instead of yellow)

We have `segment type` which is a float variable but encodes discrete values
(Maybe I'm missing something but it's not clear how to pass int variables,
maybe it's just version, haven't tried):

0) Just draw regular segment
1) Cut the first end of the segment
2) Cut the second end of the segment

```glsl
if (abs(st - 1.) < 0.01 && local_position.y < -0.5) { // in SDF space
    discard;
} else if (abs(st - 2.) < 0.01 && local_position.y > 0.5) {
    discard;
}
```

We draw a polygonal chain like this:

```Rust
for i in 0..stringline_num {
    let point = vec2(qrand::gen_range(-100., 100.), qrand::gen_range(-100., 100.));
    let segment_type = match i {
        0 => SegmentType::All,
        _ => SegmentType::NoFirst,
    };
    lines.add(Line::new(segment_type, prev, point, THICKNESS, color));
    prev = point;
}
```

This will produce cutting like a yellow line on the picture.

Our end result will look like this nice monochromatic polygonal lines:
![random lines](random_lines.png)

### Bonus: Drawing maps

We can use our lib for example to draw the map.
I've provided some OSM data [here](https://drive.google.com/file/d/16wjW3wh0f_nt9-J1ninXNTQwgeAVSawL/view?usp=sharing).
We just import all data that can be represented
as segments and voila here is the map :)

![map](map.gif)
