# Chapter 14 - Normal Mapping

In this chapter, we will explain a technique that will dramatically improve how our 3D models look like. By now we are able to apply textures to complex 3D models, but we are still far away from what real objects look like. Surfaces in the real world are not perfectly plain, they have imperfections that our 3D models currently do not have.

In order to render more realistic scenes, we are going to use normal maps. If you look at a flat surface in the real world you will see that those imperfections can be seen even at distance by the way that the light reflects on it. In a 3D scene, a flat surface will have no imperfections, we can apply a texture to it but we won’t change the way that light reflects on it. That’s the thing that makes the difference.

We may think of increasing the detail of our models by increasing the number of triangles and reflect those imperfections, but performance will degrade. What we need is a way to change the way light reflects on surfaces to increase the realism. This is achieved with the normal mapping technique.

You can find the complete source code for this chapter [here](https://github.com/lwjglgamedev/lwjglbook/tree/main/chapter-14).

## Concepts

Let’s go back to the plain surface example, a plane can be defined by two triangles which form a quad. If you remember from the lighting chapters, the element that models how light reflects are surface normals. In this case, we have a single normal for the whole surface, each fragment of the surface uses the same normal when calculating how light affects them. This is shown in the next figure.

![Surface Normals](surface_normals.png)

If we could change the normals for each fragment of the surface we could model surface imperfections to render them in a more realistic way. This is shown in the next figure.

![Fragment Normals](fragment_normals.png)

The way we are going to achieve this is by loading another texture that stores the normals for the surface. Each pixel of the normal texture will contain the values of the $$x$$, y and $$z$$ coordinates of the normal stored as an RGB value.

Let’s use the following texture to draw a quad.

![Texture](rock.png)

An example of a normal map texture for the image above may be the following.

![Normal map texture](rock_normals.png)

As you can see, it's as if we had applied a color transformation to the original texture. Each pixel stores normal information using color components. One thing that you will usually see when viewing normal maps is that the dominant colors tend to blue. This is due to the fact that normals point to the positive $$z$$ axis. The $$z$$ component will usually have a much higher value than the $$x$$ and $$y$$ ones for plain surfaces as the normal points out of the surface. Since $$x$$, $$y$$, $$z$$ coordinates are mapped to RGB, the blue component will have also a higher value.

So, to render an object using normal maps we just need an extra texture and use it while rendering fragments to get the appropriate normal value.

## Implementation

Usually, normal maps are not defined in that way, they usually are defined in the so called tangent space. The tangent space is a coordinate system that is local to each triangle of the model. In that coordinate space the $$z$$ axis always points out of the surface. This is the reason why a normal map is usually bluish, even for complex models with opposing faces. In order to handle tangent space, we need norm,als, tangent and bi-tangent vectors. We already have normal vector, the tangent and bitangent vectors are perpendicular vectors to the normal one. We need these vectors to calculate the `TBN` matrix which will allow us to use data that is in tangent space to the coordinate system we are using in our shaders. 

You can check a great tutorial on this aspect [here](https://learnopengl.com/Advanced-Lighting/Normal-Mapping)

Therefore, ye first step is to add support for normal mapping loading the `ModelLoader` class, including tangent and bitangent information. If you recall, when setting the model loading flags for assimp, we included this one: `aiProcess_CalcTangentSpace`. This flag allows to automatically calculate tangent and bitangent data.

In the `processMaterial` method we will first query for the presence of a normal map texture. If so, we load that texture and associate that texture path to the material:

```java
public class ModelLoader {
    ...
    private static Material processMaterial(AIMaterial aiMaterial, String modelDir, TextureCache textureCache) {
        ...
        try (MemoryStack stack = MemoryStack.stackPush()) {
            ...
            AIString aiNormalMapPath = AIString.calloc(stack);
            Assimp.aiGetMaterialTexture(aiMaterial, aiTextureType_NORMALS, 0, aiNormalMapPath, (IntBuffer) null,
                    null, null, null, null, null);
            String normalMapPath = aiNormalMapPath.dataString();
            if (normalMapPath != null && normalMapPath.length() > 0) {
                material.setNormalMapPath(modelDir + File.separator + new File(normalMapPath).getName());
                textureCache.createTexture(material.getNormalMapPath());
            }
            return material;
        }
    }
    ...
}
```

In the `processMesh` method we need to load also data for tangents and bitangents:

```java
public class ModelLoader {
    ...
    private static Mesh processMesh(AIMesh aiMesh) {
        ...
        float[] tangents = processTangents(aiMesh, normals);
        float[] bitangents = processBitangents(aiMesh, normals);
        ...
        return new Mesh(vertices, normals, tangents, bitangents, textCoords, indices);
    }
    ...
}
```

The `processTangents` and `processBitangents` methods are quite similar to the one that loads normals:

```java
public class ModelLoader {
    ...
    private static float[] processBitangents(AIMesh aiMesh, float[] normals) {

        AIVector3D.Buffer buffer = aiMesh.mBitangents();
        float[] data = new float[buffer.remaining() * 3];
        int pos = 0;
        while (buffer.remaining() > 0) {
            AIVector3D aiBitangent = buffer.get();
            data[pos++] = aiBitangent.x();
            data[pos++] = aiBitangent.y();
            data[pos++] = aiBitangent.z();
        }

        // Assimp may not calculate tangents with models that do not have texture coordinates. Just create empty values
        if (data.length == 0) {
            data = new float[normals.length];
        }
        return data;
    }
    ...
    private static float[] processTangents(AIMesh aiMesh, float[] normals) {

        AIVector3D.Buffer buffer = aiMesh.mTangents();
        float[] data = new float[buffer.remaining() * 3];
        int pos = 0;
        while (buffer.remaining() > 0) {
            AIVector3D aiTangent = buffer.get();
            data[pos++] = aiTangent.x();
            data[pos++] = aiTangent.y();
            data[pos++] = aiTangent.z();
        }

        // Assimp may not calculate tangents with models that do not have texture coordinates. Just create empty values
        if (data.length == 0) {
            data = new float[normals.length];
        }
        return data;
    }
    ...
}
```

As you can see, we need to modify also `Mesh` and `Material` classes to hold the new data. Let's start by the `Mesh` class:

```java
public class Mesh {
    ...
    public Mesh(float[] positions, float[] normals, float[] tangents, float[] bitangents, float[] textCoords, int[] indices) {
        ...
        // Tangents VBO
        vboId = glGenBuffers();
        vboIdList.add(vboId);
        FloatBuffer tangentsBuffer = MemoryUtil.memCallocFloat(tangents.length);
        tangentsBuffer.put(0, tangents);
        glBindBuffer(GL_ARRAY_BUFFER, vboId);
        glBufferData(GL_ARRAY_BUFFER, tangentsBuffer, GL_STATIC_DRAW);
        glEnableVertexAttribArray(2);
        glVertexAttribPointer(2, 3, GL_FLOAT, false, 0, 0);

        // Bitangents VBO
        vboId = glGenBuffers();
        vboIdList.add(vboId);
        FloatBuffer bitangentsBuffer = MemoryUtil.memCallocFloat(bitangents.length);
        bitangentsBuffer.put(0, bitangents);
        glBindBuffer(GL_ARRAY_BUFFER, vboId);
        glBufferData(GL_ARRAY_BUFFER, bitangentsBuffer, GL_STATIC_DRAW);
        glEnableVertexAttribArray(3);
        glVertexAttribPointer(3, 3, GL_FLOAT, false, 0, 0);

        // Texture coordinates VBO
        ...
        glEnableVertexAttribArray(4);
        glVertexAttribPointer(4, 2, GL_FLOAT, false, 0, 0);
        ...
        MemoryUtil.memFree(tangentsBuffer);
        MemoryUtil.memFree(bitangentsBuffer);
        ...
    }
    ...
}
```

We need to create two new VBOs for tangent and bitangent data (which follow a structure similar to the normals data) and therefore update the position of the texture coordinates VBO.

In the `Material` class we need to include the path to the normal mapping texture path:

```java
public class Material {
    ...
    private String normalMapPath;
    ...
    public String getNormalMapPath() {
        return normalMapPath;
    }
    ...
    public void setNormalMapPath(String normalMapPath) {
        this.normalMapPath = normalMapPath;
    }
    ...
}
```

Now we need to modify the shaders, starting by the scene vertex shader (`scene.vert`):

```glsl
#version 330

layout (location=0) in vec3 position;
layout (location=1) in vec3 normal;
layout (location=2) in vec3 tangent;
layout (location=3) in vec3 bitangent;
layout (location=4) in vec2 texCoord;

out vec3 outPosition;
out vec3 outNormal;
out vec3 outTangent;
out vec3 outBitangent;
out vec2 outTextCoord;

uniform mat4 projectionMatrix;
uniform mat4 viewMatrix;
uniform mat4 modelMatrix;

void main()
{
    mat4 modelViewMatrix = viewMatrix * modelMatrix;
    vec4 mvPosition =  modelViewMatrix * vec4(position, 1.0);
    gl_Position   = projectionMatrix * mvPosition;
    outPosition   = mvPosition.xyz;
    outNormal     = normalize(modelViewMatrix * vec4(normal, 0.0)).xyz;
    outTangent    = normalize(modelViewMatrix * vec4(tangent, 0)).xyz;
    outBitangent  = normalize(modelViewMatrix * vec4(bitangent, 0)).xyz;
    outTextCoord  = texCoord;
}
```

As you can see we need to define the new input data associated to bitangent and tangent. We transform those elements in the same way that we handled the normal and pass that data as an input to the fragment shader (`scene.frag`):

```glsl
#version 330
...
in vec3 outTangent;
in vec3 outBitangent;
...
struct Material
{
    vec4 ambient;
    vec4 diffuse;
    vec4 specular;
    float reflectance;
    int hasNormalMap;
};
...
uniform sampler2D normalSampler;
...
```

We start by defining the new inputs from the vertex shader, including and additional element for the `Material` struct which signals if there is a normal map available or not (`hasNormalMap`). We also add a new uniform for the normal map texture (`normalSampler`)). The next step is to define a function hat updates the normal based on normal map texture:

```glsl
...
...
vec3 calcNormal(vec3 normal, vec3 tangent, vec3 bitangent, vec2 textCoords) {
    mat3 TBN = mat3(tangent, bitangent, normal);
    vec3 newNormal = texture(normalSampler, textCoords).rgb;
    newNormal = normalize(newNormal * 2.0 - 1.0);
    newNormal = normalize(TBN * newNormal);
    return newNormal;
}

void main() {
    vec4 text_color = texture(txtSampler, outTextCoord);
    vec4 ambient = calcAmbient(ambientLight, text_color + material.ambient);
    vec4 diffuse = text_color + material.diffuse;
    vec4 specular = text_color + material.specular;

    vec3 normal = outNormal;
    if (material.hasNormalMap > 0) {
        normal = calcNormal(outNormal, outTangent, outBitangent, outTextCoord);
    }

    vec4 diffuseSpecularComp = calcDirLight(diffuse, specular, dirLight, outPosition, normal);

    for (int i=0; i<MAX_POINT_LIGHTS; i++) {
        if (pointLights[i].intensity > 0) {
            diffuseSpecularComp += calcPointLight(diffuse, specular, pointLights[i], outPosition, normal);
        }
    }

    for (int i=0; i<MAX_SPOT_LIGHTS; i++) {
        if (spotLights[i].pl.intensity > 0) {
            diffuseSpecularComp += calcSpotLight(diffuse, specular, spotLights[i], outPosition, normal);
        }
    }
    fragColor = ambient + diffuseSpecularComp;

    if (fog.activeFog == 1) {
        fragColor = calcFog(outPosition, fragColor, fog, ambientLight.color, dirLight);
    }
}
```

The `calcNormal` function takes the following parameters:

* The vertex normal.
* The vertex tangent.
* The vertex bitangent.
* The texture coordinates.

The first thing we do in that function is to calculate the TBN matrix. After that, we get the normal value form the normal map texture and use the TBN Matrix to pass from tangent space to view space. Remember that the colour we get are the normal coordinates, but since they are stored as RGB values they are contained in the range \[0, 1\]. We need to transform them to be in the range \[-1, 1\], so we just multiply by two and subtract 1.

Finally, we use that function only if the material defines a normal map texture.

We need to modify also the `SceneRender` class to create and use the new normals that we use in the shaders:

```java
public class SceneRender {
    ...
    private void createUniforms() {
        ...
        uniformsMap.createUniform("normalSampler");
        ...
        uniformsMap.createUniform("material.hasNormalMap");
        ...
    }
    public void render(Scene scene) {
        ...
        uniformsMap.setUniform("normalSampler", 1);
        ...
        for (Model model : models) {
            ...
            for (Material material : model.getMaterialList()) {
                ...
                String normalMapPath = material.getNormalMapPath();
                boolean hasNormalMapPath = normalMapPath != null;
                uniformsMap.setUniform("material.hasNormalMap", hasNormalMapPath ? 1 : 0);
                ...
                if (hasNormalMapPath) {
                    Texture normalMapTexture = textureCache.getTexture(normalMapPath);
                    glActiveTexture(GL_TEXTURE1);
                    normalMapTexture.bind();
                }
                ...
            }
        }
        ...
    }
    ...    
}
```

We need to update the sky box vertex shader because we have new vectors between normal data and texture coordinate:
```glsl
#version 330

layout (location=0) in vec3 position;
layout (location=1) in vec3 normal;
layout (location=4) in vec2 texCoord;
...
```

The last step is to update the `Main` class to show this effect. We will load two quads with and without normal maps associated to them. Also we will use left and right arrows to control light angle to show the effect.

```java
public class Main implements IAppLogic {
    ...
    public static void main(String[] args) {
        ...
        Engine gameEng = new Engine("chapter-14", new Window.WindowOptions(), main);
        ...
    }
    ...
    public void init(Window window, Scene scene, Render render) {
        String wallNoNormalsModelId = "quad-no-normals-model";
        Model quadModelNoNormals = ModelLoader.loadModel(wallNoNormalsModelId, "resources/models/wall/wall_nonormals.obj",
                scene.getTextureCache());
        scene.addModel(quadModelNoNormals);

        Entity wallLeftEntity = new Entity("wallLeftEntity", wallNoNormalsModelId);
        wallLeftEntity.setPosition(-3f, 0, 0);
        wallLeftEntity.setScale(2.0f);
        wallLeftEntity.updateModelMatrix();
        scene.addEntity(wallLeftEntity);

        String wallModelId = "quad-model";
        Model quadModel = ModelLoader.loadModel(wallModelId, "resources/models/wall/wall.obj",
                scene.getTextureCache());
        scene.addModel(quadModel);

        Entity wallRightEntity = new Entity("wallRightEntity", wallModelId);
        wallRightEntity.setPosition(3f, 0, 0);
        wallRightEntity.setScale(2.0f);
        wallRightEntity.updateModelMatrix();
        scene.addEntity(wallRightEntity);

        SceneLights sceneLights = new SceneLights();
        sceneLights.getAmbientLight().setIntensity(0.2f);
        DirLight dirLight = sceneLights.getDirLight();
        dirLight.setPosition(1, 1, 0);
        dirLight.setIntensity(1.0f);
        scene.setSceneLights(sceneLights);

        Camera camera = scene.getCamera();
        camera.moveUp(5.0f);
        camera.addRotation((float) Math.toRadians(90), 0);

        lightAngle = -35;
    }
    ...
    public void input(Window window, Scene scene, long diffTimeMillis, boolean inputConsumed) {
        if (inputConsumed) {
            return;
        }
        float move = diffTimeMillis * MOVEMENT_SPEED;
        Camera camera = scene.getCamera();
        if (window.isKeyPressed(GLFW_KEY_W)) {
            camera.moveForward(move);
        } else if (window.isKeyPressed(GLFW_KEY_S)) {
            camera.moveBackwards(move);
        }
        if (window.isKeyPressed(GLFW_KEY_A)) {
            camera.moveLeft(move);
        } else if (window.isKeyPressed(GLFW_KEY_D)) {
            camera.moveRight(move);
        }
        if (window.isKeyPressed(GLFW_KEY_LEFT)) {
            lightAngle -= 2.5f;
            if (lightAngle < -90) {
                lightAngle = -90;
            }
        } else if (window.isKeyPressed(GLFW_KEY_RIGHT)) {
            lightAngle += 2.5f;
            if (lightAngle > 90) {
                lightAngle = 90;
            }
        }

        MouseInput mouseInput = window.getMouseInput();
        if (mouseInput.isRightButtonPressed()) {
            Vector2f displVec = mouseInput.getDisplVec();
            camera.addRotation((float) Math.toRadians(-displVec.x * MOUSE_SENSITIVITY), (float) Math.toRadians(-displVec.y * MOUSE_SENSITIVITY));
        }

        SceneLights sceneLights = scene.getSceneLights();
        DirLight dirLight = sceneLights.getDirLight();
        double angRad = Math.toRadians(lightAngle);
        dirLight.getDirection().x = (float) Math.sin(angRad);
        dirLight.getDirection().y = (float) Math.cos(angRad);
    }
    ...
}    
```

The result is shown in the next figure.

![Normal mapping result](normal_mapping_result.png)

As you can see the quad that has a normal texture applied gives the impression of having more volume. Although it is, in essence, a plain surface like the other quad, you can see how the light reflects.


[Next chapter](../chapter-15/chapter-15.md)