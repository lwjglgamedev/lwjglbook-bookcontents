# Chapter 12 - Sky Box

In this chapter we will see how to create a sky box. A skybox will allow us to set a background to give the illusion that our 3D world is bigger. That background is wrapped around the camera position and covers the whole space. The technique that we are going to use here is to construct a big cube that will be displayed around the 3D scene, that is, the centre of the camera position will be the centre of the cube. The sides of that cube will be wrapped with a texture with hills, a blue sky and clouds that will be mapped in a way that the image appears to be a continuous landscape.

You can find the complete source code for this chapter [here](https://github.com/lwjglgamedev/lwjglbook/tree/main/chapter-12).

## Sky Box

The following picture depicts the skybox concept.

![Sky Box](<../.gitbook/assets/skybox (1).png>)

The process of creating a sky box can be summarized in the following steps:

* Create a big cube.
* Apply a texture to it that provides the illusion that we are seeing a giant landscape with no edges.
* Render the cube so its sides are at a far distance and its origin is located at the centre of the camera.

We will start by creating a new class named `SkyBox` with a constructor that receives the path to the 3D model which contains the sky box cube (with its texture) and a reference to the texture cache. This class will load that model and will create an `Entity` instance associated to that model. The definition of the `SkyBox` class is as follows.

```java
package org.lwjglb.engine.scene;

import org.lwjglb.engine.graph.*;

public class SkyBox {

    private Entity skyBoxEntity;
    private Model skyBoxModel;

    public SkyBox(String skyBoxModelPath, TextureCache textureCache) {
        skyBoxModel = ModelLoader.loadModel("skybox-model", skyBoxModelPath, textureCache);
        skyBoxEntity = new Entity("skyBoxEntity-entity", skyBoxModel.getId());
    }

    public Entity getSkyBoxEntity() {
        return skyBoxEntity;
    }

    public Model getSkyBoxModel() {
        return skyBoxModel;
    }
}
```

We will store a reference to the `SkyBox` class in the `Scene` class:

```java
public class Scene {
    ...
    private SkyBox skyBox;
    ...
    public SkyBox getSkyBox() {
        return skyBox;
    }
    ...
    public void setSkyBox(SkyBox skyBox) {
        this.skyBox = skyBox;
    }
    ...
}
```

The next step is to create another set of vertex and fragment shaders for the sky box. But, why not reuse the scene shaders that we already have? The answer is that, actually, the shaders that we will need are a simplified version of those shaders. For example, we will not be applying lights to the sky box. Below you can see the sky box vertex shader (`skybox.vert`).

```glsl
#version 330

layout (location=0) in vec3 position;
layout (location=1) in vec3 normal;
layout (location=2) in vec2 texCoord;

out vec2 outTextCoord;

uniform mat4 projectionMatrix;
uniform mat4 viewMatrix;
uniform mat4 modelMatrix;

void main()
{
    gl_Position = projectionMatrix * viewMatrix * modelMatrix * vec4(position, 1.0);
    outTextCoord = texCoord;
}
```

You can see that we still use the model matrix. Since we will scale the skybox, we need the model matrix. You may see some other implementations that increase the size of the cube that models the sky box at start time and do not need to multiply the model and the view matrix. We have chosen this approach because it’s more flexible and it allows us to change the size of the skybox at runtime, but you can easily switch to the other approach if you want.

The fragment shader (`skybox.frag`) is also very simple, we just get the color form a texture or from a diffuse color.

```glsl
#version 330

in vec2 outTextCoord;
out vec4 fragColor;

uniform vec4 diffuse;
uniform sampler2D txtSampler;
uniform int hasTexture;

void main()
{
    if (hasTexture == 1) {
        fragColor = texture(txtSampler, outTextCoord);
    } else {
        fragColor = diffuse;
    }
}
```

We will create a new class named `SkyBoxRender` to use those shaders and perform the render. The class starts by creating the shader program and setting up the required uniforms.

```java
package org.lwjglb.engine.graph;

import org.joml.Matrix4f;
import org.lwjglb.engine.scene.*;

import java.util.*;

import static org.lwjgl.opengl.GL20.*;
import static org.lwjgl.opengl.GL30.glBindVertexArray;

public class SkyBoxRender {

    private ShaderProgram shaderProgram;

    private UniformsMap uniformsMap;

    private Matrix4f viewMatrix;

    public SkyBoxRender() {
        List<ShaderProgram.ShaderModuleData> shaderModuleDataList = new ArrayList<>();
        shaderModuleDataList.add(new ShaderProgram.ShaderModuleData("resources/shaders/skybox.vert", GL_VERTEX_SHADER));
        shaderModuleDataList.add(new ShaderProgram.ShaderModuleData("resources/shaders/skybox.frag", GL_FRAGMENT_SHADER));
        shaderProgram = new ShaderProgram(shaderModuleDataList);
        viewMatrix = new Matrix4f();
        createUniforms();
    }
    ...
}
```

The `createUniforms` method is defined liked this:

```java
public class SkyBoxRender {
    ...
    private void createUniforms() {
        uniformsMap = new UniformsMap(shaderProgram.getProgramId());
        uniformsMap.createUniform("projectionMatrix");
        uniformsMap.createUniform("viewMatrix");
        uniformsMap.createUniform("modelMatrix");
        uniformsMap.createUniform("diffuse");
        uniformsMap.createUniform("txtSampler");
        uniformsMap.createUniform("hasTexture");
    }
    ...
}
```

We just create some uniforms to hold data we will need while rendering. The next step to create a new render method for the skybox that will be invoked in the global render method.

```java
public class SkyBoxRender {
    ...
    public void render(Scene scene) {
        SkyBox skyBox = scene.getSkyBox();
        if (skyBox == null) {
            return;
        }
        shaderProgram.bind();

        uniformsMap.setUniform("projectionMatrix", scene.getProjection().getProjMatrix());
        viewMatrix.set(scene.getCamera().getViewMatrix());
        viewMatrix.m30(0);
        viewMatrix.m31(0);
        viewMatrix.m32(0);
        uniformsMap.setUniform("viewMatrix", viewMatrix);
        uniformsMap.setUniform("txtSampler", 0);

        Model skyBoxModel = skyBox.getSkyBoxModel();
        Entity skyBoxEntity = skyBox.getSkyBoxEntity();
        TextureCache textureCache = scene.getTextureCache();
        for (Material material : skyBoxModel.getMaterialList()) {
            Texture texture = textureCache.getTexture(material.getTexturePath());
            glActiveTexture(GL_TEXTURE0);
            texture.bind();

            uniformsMap.setUniform("diffuse", material.getDiffuseColor());
            uniformsMap.setUniform("hasTexture", texture.getTexturePath().equals(TextureCache.DEFAULT_TEXTURE) ? 0 : 1);

            for (Mesh mesh : material.getMeshList()) {
                glBindVertexArray(mesh.getVaoId());

                uniformsMap.setUniform("modelMatrix", skyBoxEntity.getModelMatrix());
                glDrawElements(GL_TRIANGLES, mesh.getNumVertices(), GL_UNSIGNED_INT, 0);
            }
        }

        glBindVertexArray(0);

        shaderProgram.unbind();
    }
}
```

You will see that we are modifying the view matrix prior to loading that data in the associated uniform. Remember that when we move the camera, what we are actually doing is moving the whole world. So if we just multiply the view matrix as it is, the skybox will be displaced when the camera moves. But we do not want this, we want to stick it at the origin coordinates at (0, 0, 0). This is achieved by setting to 0 the parts of the view matrix that contain the translation increments (the `m30`, `m31` and `m32` components). You may think that you could avoid using the view matrix at all since the sky box must be fixed at the origin. In that case, you would see that the skybox does not rotate with the camera, which is not what we want. We need it to rotate but not translate. To render the skybox, we just set up the uniforms and render the cube associated to the sky box.

Finally, we define a `cleanup` method to properly free resources:

```java
public class SkyBoxRender {
    ...
    public void cleanup() {
        shaderProgram.cleanup();
    }
    ...
}
```

In the `Render` class we just need to instantiate the `SkyBoxRender` class and invoke the render method:

```java
public class Render {
    ...
    private SkyBoxRender skyBoxRender;
    ...

    public Render(Window window) {
        ...
        skyBoxRender = new SkyBoxRender();
    }
    
    public void render(Window window, Scene scene) {
        glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
        glViewport(0, 0, window.getWidth(), window.getHeight());

        skyBoxRender.render(scene);
        sceneRender.render(scene);
        guiRender.render(scene);
    }
    ...
}
```

You can see that we render the sky box first. This is due to the fact that if we have 3D models in the scene with transparencies we want them to be blended with the skybox (not with a black background).

Finally, in the `Main` class we just set up the sky box in the scene and create a set of tiles to give the illusion of an infinite terrain. We set up a chunk of tiles that move along with the camera position to always be shown.

```java
public class Main implements IAppLogic {
    ...
    private static final int NUM_CHUNKS = 4;

    private Entity[][] terrainEntities;

    public static void main(String[] args) {
        ...
        Engine gameEng = new Engine("chapter-12", new Window.WindowOptions(), main);
        ...
    }
    ...

    @Override
    public void init(Window window, Scene scene, Render render) {
        String quadModelId = "quad-model";
        Model quadModel = ModelLoader.loadModel("quad-model", "resources/models/quad/quad.obj",
                scene.getTextureCache());
        scene.addModel(quadModel);

        int numRows = NUM_CHUNKS * 2 + 1;
        int numCols = numRows;
        terrainEntities = new Entity[numRows][numCols];
        for (int j = 0; j < numRows; j++) {
            for (int i = 0; i < numCols; i++) {
                Entity entity = new Entity("TERRAIN_" + j + "_" + i, quadModelId);
                terrainEntities[j][i] = entity;
                scene.addEntity(entity);
            }
        }

        SceneLights sceneLights = new SceneLights();
        sceneLights.getAmbientLight().setIntensity(0.2f);
        scene.setSceneLights(sceneLights);

        SkyBox skyBox = new SkyBox("resources/models/skybox/skybox.obj", scene.getTextureCache());
        skyBox.getSkyBoxEntity().setScale(50);
        skyBox.getSkyBoxEntity().updateModelMatrix();
        scene.setSkyBox(skyBox);

        scene.getCamera().moveUp(0.1f);

        updateTerrain(scene);
    }

    @Override
    public void input(Window window, Scene scene, long diffTimeMillis, boolean inputConsumed) {
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

        MouseInput mouseInput = window.getMouseInput();
        if (mouseInput.isRightButtonPressed()) {
            Vector2f displVec = mouseInput.getDisplVec();
            camera.addRotation((float) Math.toRadians(-displVec.x * MOUSE_SENSITIVITY), (float) Math.toRadians(-displVec.y * MOUSE_SENSITIVITY));
        }
    }

    @Override
    public void update(Window window, Scene scene, long diffTimeMillis) {
        updateTerrain(scene);
    }

    public void updateTerrain(Scene scene) {
        int cellSize = 10;
        Camera camera = scene.getCamera();
        Vector3f cameraPos = camera.getPosition();
        int cellCol = (int) (cameraPos.x / cellSize);
        int cellRow = (int) (cameraPos.z / cellSize);

        int numRows = NUM_CHUNKS * 2 + 1;
        int numCols = numRows;
        int zOffset = -NUM_CHUNKS;
        float scale = cellSize / 2.0f;
        for (int j = 0; j < numRows; j++) {
            int xOffset = -NUM_CHUNKS;
            for (int i = 0; i < numCols; i++) {
                Entity entity = terrainEntities[j][i];
                entity.setScale(scale);
                entity.setPosition((cellCol + xOffset) * 2.0f, 0, (cellRow + zOffset) * 2.0f);
                entity.getModelMatrix().identity().scale(scale).translate(entity.getPosition());
                xOffset++;
            }
            zOffset++;
        }
    }
}
```

[Next chapter](../chapter-13/chapter-13.md)
