# Lab 3: Shaders and Lighting

Lab 3 covers writing vertex and fragment shaders in WebGL (GLSL ES 1.0) with gl-matrix.

- **3.1**: set up a full WebGL pipeline (buffers, attribute/uniform locations, perspective and modelview matrices, `drawElements`) to render an indexed cube, then colour each vertex and make the cube spin.
- **3.2**: Lambert shading in the vertex shader, then cell (toon) shading in the fragment shader by quantizing each colour channel with `cellSize`.
- **3.3**: optional free-form shader.

## Shader Dojo (practice tool)

`shader-dojo.html` is a self-contained practice page for the Lab 3 test. Open it in Chrome, Edge or Firefox; no install or server needed.

- **Lab 3.1 lessons**: the handout template with one section blanked out per lesson, ending with the full empty template and Exercise 3.1 (colour + motion).
- **Lab 3.2 lessons**: Lambert, cell shading, and a per-fragment (Phong) bonus, run inside the real Lab 3.2 page.
- **Lab 3.3**: a free shader with ideas.
- **GLSL gym**: extra shader exercises (normals, lighting, materials, noise, vertex animation).

Each lesson explains a concept, gives a task, runs your code live next to the target, and checks your answer pixel by pixel. Progress is saved in your browser.
