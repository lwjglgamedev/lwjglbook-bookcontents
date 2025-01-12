# Chapter 07 - Textures

In this chapter we will learn how to load textures, how they relate to a model and how to use them in the rendering process

You can find the complete source code for this chapter [here](https://github.com/lwjglgamedev/lwjglbook/tree/main/chapter-07).

## Texture loading

A texture is an image which is mapped to a model to set the color of the pixels of the model. You can think of a texture as a skin that is wrapped around your 3D model. What you do is assign points in the image texture to the vertices in your model. With that information OpenGL is able to calculate the color to apply to the other pixels based on the texture image.

![Texture mapping](texture_mapping.png)

The texture image does not have to be the same size as the model. It can be larger or smaller. OpenGL will extrapolate the color if the pixel to be processed cannot be mapped to a specific point in the texture. You can control how this process is done when a specific texture is created.

So basically what we must do, in order to apply a texture to a model, is assigning texture coordinates to each of our vertices. The texture coordinate system is a bit different from the coordinate system of our model. First of all, we have a 2D texture so our coordinates will only have two components, x and y. Besides that, the origin is setup in the top left corner of the image and the maximum value of the x or y value is 1.

![Texture coordinates](texture_coordinates.png)

How do we relate texture coordinates with our position coordinates? Easy, in the same way we passed the color information. We set up a VBO which will have a texture coordinate for each vertex position.

So let’s start modifying the code base to use textures in our 3D cube. The first step is to load the image that will be used as a texture. For this task, we will use the LWJGL wrapper for the [stb](https://github.com/nothings/stb) library. In order to do that,  we need first to declare that dependency, including the natives in our `pom.xml` file.

```xml
<dependency>
    <groupId>org.lwjgl</groupId>
    <artifactId>lwjgl-stb</artifactId>
    <version>${lwjgl.version}</version>
</dependency>
[...]
<dependency>
    <groupId>org.lwjgl</groupId>
    <artifactId>lwjgl-stb</artifactId>
    <version>${lwjgl.version}</version>
    <classifier>${native.target}</classifier>
    <scope>runtime</scope>
</dependency>
```

The first step we will do is to create a new `Texture` class that will perform all the necessary steps to load a texture and is defined like this:
```java
package org.lwjglb.engine.graph;

import org.lwjgl.system.MemoryStack;

import java.nio.*;

import static org.lwjgl.opengl.GL30.*;
import static org.lwjgl.stb.STBImage.*;

public class Texture {

    private int textureId;
    private String texturePath;

    public Texture(int width, int height, ByteBuffer buf) {
        this.texturePath = "";
        generateTexture(width, height, buf);
    }

    public Texture(String texturePath) {
        try (MemoryStack stack = MemoryStack.stackPush()) {
            this.texturePath = texturePath;
            IntBuffer w = stack.mallocInt(1);
            IntBuffer h = stack.mallocInt(1);
            IntBuffer channels = stack.mallocInt(1);

            ByteBuffer buf = stbi_load(texturePath, w, h, channels, 4);
            if (buf == null) {
                throw new RuntimeException("Image file [" + texturePath + "] not loaded: " + stbi_failure_reason());
            }

            int width = w.get();
            int height = h.get();

            generateTexture(width, height, buf);

            stbi_image_free(buf);
        }
    }

    public void bind() {
        glBindTexture(GL_TEXTURE_2D, textureId);
    }

    public void cleanup() {
        glDeleteTextures(textureId);
    }

    private void generateTexture(int width, int height, ByteBuffer buf) {
        textureId = glGenTextures();

        glBindTexture(GL_TEXTURE_2D, textureId);
        glPixelStorei(GL_UNPACK_ALIGNMENT, 1);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
        glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, width, height, 0,
                GL_RGBA, GL_UNSIGNED_BYTE, buf);
        glGenerateMipmap(GL_TEXTURE_2D);
    }

    public String getTexturePath() {
        return texturePath;
    }
}
```

The first thing we do in the constructor is to allocate `IntBuffer`s for the library to return the image size and number of channels. Then, we call the `stbi_load` method to actually load the image into a `ByteBuffer`. This method requires the following parameters:

* `filePath`: The absolute path to the file. The stb library is native and does not understand anything about `CLASSPATH`. Therefore, we will be using regular file system paths.
* `width`:  Image width. This will be populated with the image width.
* `height`: Image height. This will be populated with the image height.
* `channels`: The image channels.
* `desired_channels`: The desired image channels. We pass 4 (RGBA).

One important thing to remember is that OpenGL, for historical reasons, requires that texture images have a size \(number of texels in each dimension\) of a power of two \(2, 4, 8, 16, ....\). I think this is not required by OpenGL drivers anymore but if you have some issues you can try modifying the dimensions.

The next step is to upload the texture into the GPU. This will be done in the `generateTexture` method. First of all we need to create a new texture identifier (by calling the `glGenTextures` function). After that we need to bind to that texture (by calling the `glBindTexture`). Then we need to tell OpenGL how to unpack our RGBA bytes. Since each component is one byte in size we just use `GL_UNPACK_ALIGNMENT` for the `glPixelStorei` function. Finally we load the texture data by calling the `glTexImage2D`.

The `glTexImage2D` method has the following parameters:

* `target`: Specifies the target texture \(its type\). In this case: `GL_TEXTURE_2D`. 
* `level`: Specifies the level-of-detail number. Level 0 is the base image level. Level n is the nth mipmap reduction image. More on this later.
* `internal format`: Specifies the number of colour components in the texture.
* `width`: Specifies the width of the texture image.
* `height`: Specifies the height of the texture image.
* `border`: This value must be zero.
* `format`: Specifies the format of the pixel data: RGBA in this case.
* `type`: Specifies the data type of the pixel data. We are using unsigned bytes for this.
* `data`: The buffer that stores our data.

After that, by calling the `glTexParameteri` function we basically say that when a pixel is drawn with no direct one to one association to a texture coordinate it will pick the nearest texture coordinate point. After that, we generate a mipmap. A mipmap is a decreasing resolution set of images generated from a high detailed texture. These lower resolution images will be used automatically when our object is scaled. We do this when calling the `glGenerateMipmap` function. And that’s all, we have successfully loaded our texture. Now we need to use it.


Now we will create a texture cache. It is very frequent that models reuse the same texture, therefore, instead of loading the same texture multiple times, we will cache the textures already loaded to load each texture just once. This will be controlled by the `TextureCache` class:

```java
package org.lwjglb.engine.graph;

import java.util.*;

public class TextureCache {

    public static final String DEFAULT_TEXTURE = "resources/models/default/default_texture.png";

    private Map<String, Texture> textureMap;

    public TextureCache() {
        textureMap = new HashMap<>();
        textureMap.put(DEFAULT_TEXTURE, new Texture(DEFAULT_TEXTURE));
    }

    public void cleanup() {
        textureMap.values().forEach(Texture::cleanup);
    }

    public Texture createTexture(String texturePath) {
        return textureMap.computeIfAbsent(texturePath, Texture::new);
    }

    public Texture getTexture(String texturePath) {
        Texture texture = null;
        if (texturePath != null) {
            texture = textureMap.get(texturePath);
        }
        if (texture == null) {
            texture = textureMap.get(DEFAULT_TEXTURE);
        }
        return texture;
    }
}
```

As you can see we just store te loaded textures in a `Map` and return a default texture in case texture path is null (models with no textures). The default texture is just a back image which can be combined with model which do not define textures but colors so we can combine both in the fragment shader. The `TextureCache` class instance will be stored in the `Scene` class:
```java
public class Scene {
    ...
    private TextureCache textureCache;
    ...
    public Scene(int width, int height) {
        ...
        textureCache = new TextureCache();
    }
    ...
    public TextureCache getTextureCache() {
        return textureCache;
    }
    ...
}
```

Now we new to change the way we define models to add support for textures. In order to do so, and to prepare for the more complex models we are going to load in next chapters, we will introduce a new class named `Material`. This class will hold texture path and a list of `Mesh` instances. Therefore, we will associated `Model` instances with a `List` of `Material`'s instead of `Mesh`'es. In next chapters materials will be able to contain other properties, such as diffuse or specular colors.

The `Material` class is defined like this:
```java
package org.lwjglb.engine.graph;

import java.util.*;

public class Material {

    private List<Mesh> meshList;
    private String texturePath;

    public Material() {
        meshList = new ArrayList<>();
    }

    public void cleanup() {
        meshList.forEach(Mesh::cleanup);
    }

    public List<Mesh> getMeshList() {
        return meshList;
    }

    public String getTexturePath() {
        return texturePath;
    }

    public void setTexturePath(String texturePath) {
        this.texturePath = texturePath;
    }
}
```

As you can see, `Mesh` instances are now under the `Material` class. Therefore, we need to modify the `Model` class like this:
```java
package org.lwjglb.engine.graph;

import org.lwjglb.engine.scene.Entity;

import java.util.*;

public class Model {

    private final String id;
    private List<Entity> entitiesList;
    private List<Material> materialList;

    public Model(String id, List<Material> materialList) {
        this.id = id;
        entitiesList = new ArrayList<>();
        this.materialList = materialList;
    }

    public void cleanup() {
        materialList.forEach(Material::cleanup);
    }

    public List<Entity> getEntitiesList() {
        return entitiesList;
    }

    public String getId() {
        return id;
    }

    public List<Material> getMaterialList() {
        return materialList;
    }
}
```

As we said before we need to pass texture coordinates as another VBO. So we will modify our `Mesh` class to accept an array of floats that contains texture coordinates instead of colors. The `Mesh` class is modified like this:

```java
public class Mesh {
    ...
    public Mesh(float[] positions, float[] textCoords, int[] indices) {
        numVertices = indices.length;
        vboIdList = new ArrayList<>();

        vaoId = glGenVertexArrays();
        glBindVertexArray(vaoId);

        // Positions VBO
        int vboId = glGenBuffers();
        vboIdList.add(vboId);
        FloatBuffer positionsBuffer = MemoryUtil.memCallocFloat(positions.length);
        positionsBuffer.put(0, positions);
        glBindBuffer(GL_ARRAY_BUFFER, vboId);
        glBufferData(GL_ARRAY_BUFFER, positionsBuffer, GL_STATIC_DRAW);
        glEnableVertexAttribArray(0);
        glVertexAttribPointer(0, 3, GL_FLOAT, false, 0, 0);

        // Texture coordinates VBO
        vboId = glGenBuffers();
        vboIdList.add(vboId);
        FloatBuffer textCoordsBuffer = MemoryUtil.memCallocFloat(textCoords.length);
        textCoordsBuffer.put(0, textCoords);
        glBindBuffer(GL_ARRAY_BUFFER, vboId);
        glBufferData(GL_ARRAY_BUFFER, textCoordsBuffer, GL_STATIC_DRAW);
        glEnableVertexAttribArray(1);
        glVertexAttribPointer(1, 2, GL_FLOAT, false, 0, 0);

        // Index VBO
        vboId = glGenBuffers();
        vboIdList.add(vboId);
        IntBuffer indicesBuffer = MemoryUtil.memCallocInt(indices.length);
        indicesBuffer.put(0, indices);
        glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, vboId);
        glBufferData(GL_ELEMENT_ARRAY_BUFFER, indicesBuffer, GL_STATIC_DRAW);

        glBindBuffer(GL_ARRAY_BUFFER, 0);
        glBindVertexArray(0);

        MemoryUtil.memFree(positionsBuffer);
        MemoryUtil.memFree(textCoordsBuffer);
        MemoryUtil.memFree(indicesBuffer);
    }
    ...
}
```

## Using the textures

Now we need to use the texture in our shaders. In the vertex shader we have changed the second input parameter because now it’s a `vec2` \(we also changed the parameter name). The vertex shader, as in the color case, just passes the texture coordinates to be used by the fragment shader.

```glsl
#version 330

layout (location=0) in vec3 position;
layout (location=1) in vec2 texCoord;

out vec2 outTextCoord;

uniform mat4 projectionMatrix;
uniform mat4 modelMatrix;

void main()
{
    gl_Position = projectionMatrix * modelMatrix * vec4(position, 1.0);
    outTextCoord = texCoord;
}
```

In the fragment shader we must use the texture coordinates in order to set the pixel colors by sampling a texture (through a `sampler2D` uniform)

```glsl
#version 330

in vec2 outTextCoord;

out vec4 fragColor;

uniform sampler2D txtSampler;

void main()
{
    fragColor = texture(txtSampler, outTextCoord);
}
```

We will see now how all of this is used in the `SceneRender` class. First, we need to create a new uniform for the texture sampler.

```java
public class SceneRender {
    ...
    private void createUniforms() {
        ...
        uniformsMap.createUniform("txtSampler");
    }
    ...
}
```

Now, we can use the texture in the render process:

```java
public class SceneRender {
    ...
    public void render(Scene scene) {
        shaderProgram.bind();

        uniformsMap.setUniform("projectionMatrix", scene.getProjection().getProjMatrix());

        uniformsMap.setUniform("txtSampler", 0);

        Collection<Model> models = scene.getModelMap().values();
        TextureCache textureCache = scene.getTextureCache();
        for (Model model : models) {
            List<Entity> entities = model.getEntitiesList();

            for (Material material : model.getMaterialList()) {
                Texture texture = textureCache.getTexture(material.getTexturePath());
                glActiveTexture(GL_TEXTURE0);
                texture.bind();

                for (Mesh mesh : material.getMeshList()) {
                    glBindVertexArray(mesh.getVaoId());
                    for (Entity entity : entities) {
                        uniformsMap.setUniform("modelMatrix", entity.getModelMatrix());
                        glDrawElements(GL_TRIANGLES, mesh.getNumVertices(), GL_UNSIGNED_INT, 0);
                    }
                }
            }
        }

        glBindVertexArray(0);

        shaderProgram.unbind();
    }
}
```

As you can see, we first set the texture sampler uniform to the `0` value. Let's explain why we do this. A graphics card has several spaces or slots to store textures. Each of these spaces is called a texture unit. When we are working with textures we must set the texture unit that we want to work with. In this case we are using just one texture, so we will use the texture unit `0`. The uniform has a `sampler2D` type and will hold the value of the texture unit that we want to work with.. When we iterate over models and materials we get the texture associated to each material from the cache, activate the texture unit by calling the `glActiveTexture` function with the parameter `GL_TEXTURE0` and bind it. This is the way we relate the texture unit and the texture identifier.

We need to modify also the `UniformsMap` class to add a new method which accepts an integer to set up the sampler value, which will be called also `setUniform` but accepting the name of the uniform and an integer value. Since we will be repeating some code between the `setUniform` method used to set up matrices and this new one, we will extract the part of the code that retrieves the uniform location to a new method named `getUniformLocation`. The changes in the `UniformsMap` class are shown below:

```java
public class UniformsMap {
    ...
    private int getUniformLocation(String uniformName) {
        Integer location = uniforms.get(uniformName);
        if (location == null) {
            throw new RuntimeException("Could not find uniform [" + uniformName + "]");
        }
        return location.intValue();
    }

    public void setUniform(String uniformName, int value) {
        glUniform1i(getUniformLocation(uniformName), value);
    }

    public void setUniform(String uniformName, Matrix4f value) {
        try (MemoryStack stack = MemoryStack.stackPush()) {
            glUniformMatrix4fv(getUniformLocation(uniformName), false, value.get(stack.mallocFloat(16)));
        }
    }
    ...
}
```

Right now, we have just modified our code base to support textures. Now we need to setup texture coordinates for our 3D cube. Our texture image file will be something like this:

![Cube texture](cube_texture.png)

In our 3D model we have eight vertices. Let’s see how this can be done. Let’s first define the front face texture coordinates for each vertex.

![Texture coordinates front face](cube_texture_front_face.png)

| Vertex | Texture Coordinate |
| --- | --- |
| V0 | \(0.0, 0.0\) |
| V1 | \(0.0, 0.5\) |
| V2 | \(0.5, 0.5\) |
| V3 | \(0.5, 0.0\) |

Now, let’s define the texture mapping of the top face.

![Texture coordinates top face](cube_texture_top_face.png)

| Vertex | Texture Coordinate |
| --- | --- |
| V4 | \(0.0, 0.5\) |
| V5 | \(0.5, 0.5\) |
| V0 | \(0.0, 1.0\) |
| V3 | \(0.5, 1.0\) |

As you can see we have a problem, we need to setup different texture coordinates for the same vertices \(V0 and V3\). How can we solve this? The only way to solve it is to repeat some vertices and associate different texture coordinates. For the top face we need to repeat the four vertices and assign them the correct texture coordinates.

Since the front, back and lateral faces use the same texture we will not need to repeat all of these vertices. You have the complete definition in the source code, but we needed to move from 8 points to 20. 

In the next chapters we will learn how to load models generated by 3D modeling tools so we won’t need to define by hand the positions and texture coordinates \(which by the way, would be impractical for more complex models\).

We just need to modify the `init` method in the `Main` class to define the texture coordinates and load texture data:

```java
public class Main implements IAppLogic {
    ...
    public static void main(String[] args) {
        ...
        Engine gameEng = new Engine("chapter-07", new Window.WindowOptions(), main);
        ...
    }
    ...
    public void init(Window window, Scene scene, Render render) {
        float[] positions = new float[]{
                // V0
                -0.5f, 0.5f, 0.5f,
                // V1
                -0.5f, -0.5f, 0.5f,
                // V2
                0.5f, -0.5f, 0.5f,
                // V3
                0.5f, 0.5f, 0.5f,
                // V4
                -0.5f, 0.5f, -0.5f,
                // V5
                0.5f, 0.5f, -0.5f,
                // V6
                -0.5f, -0.5f, -0.5f,
                // V7
                0.5f, -0.5f, -0.5f,

                // For text coords in top face
                // V8: V4 repeated
                -0.5f, 0.5f, -0.5f,
                // V9: V5 repeated
                0.5f, 0.5f, -0.5f,
                // V10: V0 repeated
                -0.5f, 0.5f, 0.5f,
                // V11: V3 repeated
                0.5f, 0.5f, 0.5f,

                // For text coords in right face
                // V12: V3 repeated
                0.5f, 0.5f, 0.5f,
                // V13: V2 repeated
                0.5f, -0.5f, 0.5f,

                // For text coords in left face
                // V14: V0 repeated
                -0.5f, 0.5f, 0.5f,
                // V15: V1 repeated
                -0.5f, -0.5f, 0.5f,

                // For text coords in bottom face
                // V16: V6 repeated
                -0.5f, -0.5f, -0.5f,
                // V17: V7 repeated
                0.5f, -0.5f, -0.5f,
                // V18: V1 repeated
                -0.5f, -0.5f, 0.5f,
                // V19: V2 repeated
                0.5f, -0.5f, 0.5f,
        };
        float[] textCoords = new float[]{
                0.0f, 0.0f,
                0.0f, 0.5f,
                0.5f, 0.5f,
                0.5f, 0.0f,

                0.0f, 0.0f,
                0.5f, 0.0f,
                0.0f, 0.5f,
                0.5f, 0.5f,

                // For text coords in top face
                0.0f, 0.5f,
                0.5f, 0.5f,
                0.0f, 1.0f,
                0.5f, 1.0f,

                // For text coords in right face
                0.0f, 0.0f,
                0.0f, 0.5f,

                // For text coords in left face
                0.5f, 0.0f,
                0.5f, 0.5f,

                // For text coords in bottom face
                0.5f, 0.0f,
                1.0f, 0.0f,
                0.5f, 0.5f,
                1.0f, 0.5f,
        };
        int[] indices = new int[]{
                // Front face
                0, 1, 3, 3, 1, 2,
                // Top Face
                8, 10, 11, 9, 8, 11,
                // Right face
                12, 13, 7, 5, 12, 7,
                // Left face
                14, 15, 6, 4, 14, 6,
                // Bottom face
                16, 18, 19, 17, 16, 19,
                // Back face
                4, 6, 7, 5, 4, 7,};
        Texture texture = scene.getTextureCache().createTexture("resources/models/cube/cube.png");
        Material material = new Material();
        material.setTexturePath(texture.getTexturePath());
        List<Material> materialList = new ArrayList<>();
        materialList.add(material);

        Mesh mesh = new Mesh(positions, textCoords, indices);
        material.getMeshList().add(mesh);
        Model cubeModel = new Model("cube-model", materialList);
        scene.addModel(cubeModel);

        cubeEntity = new Entity("cube-entity", cubeModel.getId());
        cubeEntity.setPosition(0, 0, -2);
        scene.addEntity(cubeEntity);
    }
```

The final result is like this.

![Cube with texture](cube_with_texture.png)

[Next chapter](../chapter-08/chapter-08.md)
