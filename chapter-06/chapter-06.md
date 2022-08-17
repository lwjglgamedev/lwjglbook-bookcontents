# Chapter 06 - Going 3D

In this chapter we will set up the basis for 3D models and explain the concept of model transformations to render our first 3D shaper, a rotating cube.

You can find the complete source code for this chapter [here](https://github.com/lwjglgamedev/lwjglbook/chapter-06).

## Models and Entities

Let us first define the concept of a 3D model. Up to now, we have been working with meshes (a collection of vertices). A model is an structure which glues together vertices, colors, textures and materials. A model may be composed of several meshes and cane be used by several game entities. A game entity represents a player and enemy, and obstacle, anything that is part of the 3D scene. In this book we will assume that an entity always is related to a model (although you can have entities that are not rendered and therefore cannot have a model). An entity has specific data such a position, which we need to use when rendering. You will see later on that we start the render process by getting the models and then draw the entities associated to that model. This is because efficiency, since several entities can share the same model is better to set up the elements that belong to the model once and later on handle the data that is specific for each entity.

The class that represents a model is, obviously, called `Model` and is defined like this:

```java
package org.lwjglb.engine.graph;

import org.lwjglb.engine.scene.Entity;

import java.util.*;

public class Model {

    private final String id;
    private List<Entity> entitiesList;
    private List<Mesh> meshList;

    public Model(String id, List<Mesh> meshList) {
        this.id = id;
        this.meshList = meshList;
        entitiesList = new ArrayList<>();
    }

    public void cleanup() {
        meshList.stream().forEach(Mesh::cleanup);
    }

    public List<Entity> getEntitiesList() {
        return entitiesList;
    }

    public String getId() {
        return id;
    }

    public List<Mesh> getMeshList() {
        return meshList;
    }

}
```

As you can see, a model, by now, stores a list of `Mesh` instances an has a unique identifier. In addition to that, we store the list of game entities (modelled by the `Entity` class) that are associated to that model. If you are going to create a full engine, you may want to store those relationships in somewhere else (not in the model), however, for simplicity we will store those links in the `Model` class. This way, render process will be simpler.

Prior to see what an `Entity` looks like, let us discuss a little bit about model transformations. In order to represent any model in a 3D scene we need to provide support for some basic operations to act upon any model:

* Translation: Move an object by some amount on any of the three axes.
* Rotation: Rotate an object by some amount of degrees around any of the three axes.
* Scale: Adjust the size of an object.

![Transformations](transformations.png)

The operations described above are known as transformations. And you are probably guessing, correctly, that the way we are going to achieve that is by multiplying our coordinates by a set of matrices \(one for translation, one for rotation and one for scaling\). Those three matrices will be combined into a single matrix called world matrix and passed as a uniform to our vertex shader.

The reason why it is called world matrix is because we are transforming from model coordinates to world coordinates. When you learn about loading 3D models you will see that those models are defined in their own coordinate systems. They don’t know the size of your 3D space and they need to be placed in it. So when we multiply our coordinates by our matrix what we are doing is transforming from one coordinate system \(the model one\) to another \(the one for our 3D world\).

That world matrix will be calculated like this \(the order is important since multiplication using matrices is not commutative\):

$$
World Matrix=\left[Translation Matrix\right]\left[Rotation Matrix\right]\left[Scale Matrix\right]
$$

If we include our projection matrix in the transformation matrix it would be like this:

$$
\begin{array}{lcl}
Transf & = & \left[Proj Matrix\right]\left[Translation Matrix\right]\left[Rotation Matrix\right]\left[Scale Matrix\right] \\
 & = & \left[Proj Matrix\right]\left[World Matrix\right]
\end{array}
$$

The translation matrix is defined like this:

$$
\begin{bmatrix}
1 & 0 & 0 & dx \\
0 & 1 & 0 & dy \\
0 & 0 & 1 & dz \\
0 & 0 & 0 & 1
\end{bmatrix}
$$


Translation Matrix Parameters:

* dx: Displacement along the x axis.
* dy: Displacement along the y axis.
* dz: Displacement along the z axis.

The scale matrix is defined like this:

$$
\begin{bmatrix}
sx & 0 & 0 & 0 \\
0 & sy & 0 & 0 \\
0 & 0 & sz & 0 \\
0 & 0 & 0 & 1
\end{bmatrix}
$$


Scale Matrix Parameters:

* sx: Scaling along the x axis.
* sy: Scaling along the y axis.
* sz: Scaling along the z axis.

The rotation matrix is much more complex. But keep in mind that it can be constructed by the multiplication of 3 rotation matrices for a single axis, each or by applying a quaternion (more on this later).

So let us define the `Entity` class:
```java
package org.lwjglb.engine.scene;

import org.joml.*;

public class Entity {

    private final String id;
    private final String modelId;
    private Matrix4f modelMatrix;
    private Vector3f position;
    private Quaternionf rotation;
    private float scale;

    public Entity(String id, String modelId) {
        this.id = id;
        this.modelId = modelId;
        modelMatrix = new Matrix4f();
        position = new Vector3f();
        rotation = new Quaternionf();
        scale = 1;
    }

    public String getId() {
        return id;
    }

    public String getModelId() {
        return modelId;
    }

    public Matrix4f getModelMatrix() {
        return modelMatrix;
    }

    public Vector3f getPosition() {
        return position;
    }

    public Quaternionf getRotation() {
        return rotation;
    }

    public float getScale() {
        return scale;
    }

    public final void setPosition(float x, float y, float z) {
        position.x = x;
        position.y = y;
        position.z = z;
    }

    public void setRotation(float x, float y, float z, float angle) {
        this.rotation.fromAxisAngleRad(x, y, z, angle);
    }

    public void setScale(float scale) {
        this.scale = scale;
    }

    public void updateModelMatrix() {
        modelMatrix.translationRotateScale(position, rotation, scale);
    }
}
```

As you can see a `Model` instance also has a unique identifier and defines attributes for its position (as a 3 components vector), its scale (just a float, we will be assuming that we scale evenly across all three axis) and rotation (as a quaternion). We could have store rotation information storing rotation angles for pitch, yaw and roll,. but instead we are using a strange mathematical artifact, named quaternion, that you may not have heard of. The problem with using rotation angles is the so called gimbal lock. When applying rotation using these angles (called Euler angles) we may end up aligning two rotating axis and losing degrees of freedom, resulting in the inability to properly rotate an object. Quaternions do not have these problem. Instead of me trying to poorly explain what quaternions are, le mw just link to an excellent [blog entry](https://www.3dgep.com/understanding-quaternions/) which explain all the concepts behind them. If you do not want to deep dive into them, just keep in mind that they will allow to express rotations without the problems of Euler angles.

All the transformations applied to a model are defined by a 4x4 matrix, therefore a `Model` instance stores a `Matrix4f` instance which is automatically constructed by JOML method `translationRotateScale` using position, scale and rotation. We will need to call the `updateModelMatrix` method each time we modify the attributes of a `Model` instance to update that matrix.
 
## Other code changes

We need to change the `Scene` class to store models instead of directly `Mesh` instances. In addition to that, we need to add support for linking `Entity` instances with models so we can later on render them.

```java
package org.lwjglb.engine.scene;

import org.lwjglb.engine.graph.Model;

import java.util.*;

public class Scene {

    private Map<String, Model> modelMap;
    private Projection projection;

    public Scene(int width, int height) {
        modelMap = new HashMap<>();
        projection = new Projection(width, height);
    }

    public void addEntity(Entity entity) {
        String modelId = entity.getModelId();
        Model model = modelMap.get(modelId);
        if (model == null) {
            throw new RuntimeException("Could not find model [" + modelId + "]");
        }
        model.getEntitiesList().add(entity);
    }

    public void addModel(Model model) {
        modelMap.put(model.getId(), model);
    }

    public void cleanup() {
        modelMap.values().stream().forEach(Model::cleanup);
    }

    public Map<String, Model> getModelMap() {
        return modelMap;
    }

    public Projection getProjection() {
        return projection;
    }

    public void resize(int width, int height) {
        projection.updateProjMatrix(width, height);
    }
}
```

Now we need to modify a little it the `SceneRender` class. Ths first thing that we need to do is to pass model matrix information to the shader through an uniform. Therefore, we will create a new uniform named `modelMatrix` in the vertex shader and, consequently, retrieve its location in the `createUniforms` method.
```java
public class SceneRender {
    ...
    private void createUniforms() {
        ...
        uniformsMap.createUniform("modelMatrix");
    }
    ...
}
```

The next step is to modify the `render` method to change how we access models and set up properly the model matrix uniform:
```java
public class SceneRender {
    ...
    public void render(Scene scene) {
        shaderProgram.bind();

        uniformsMap.setUniform("projectionMatrix", scene.getProjection().getProjMatrix());

        Collection<Model> models = scene.getModelMap().values();
        for (Model model : models) {
            model.getMeshList().stream().forEach(mesh -> {
                glBindVertexArray(mesh.getVaoId());
                List<Entity> entities = model.getEntitiesList();
                for (Entity entity : entities) {
                    uniformsMap.setUniform("modelMatrix", entity.getModelMatrix());
                    glDrawElements(GL_TRIANGLES, mesh.getNumVertices(), GL_UNSIGNED_INT, 0);
                }
            });
        }

        glBindVertexArray(0);

        shaderProgram.unbind();
    }
    ...
}
```

As you can see, we iterate over the models, then over their meshes, binding their VAO and after that we get the associated entities. For each entity, prior to invoking the drawing call, we fill up the `modelMatrix` uniform with the proper data.

The next step is to modify the vertex shader to use the `modelMatrix` uniform.
```glsl
#version 330

layout (location=0) in vec3 position;
layout (location=1) in vec3 color;

out vec3 outColor;

uniform mat4 projectionMatrix;
uniform mat4 modelMatrix;

void main()
{
    gl_Position = projectionMatrix * modelMatrix * vec4(position, 1.0);
    outColor = color;
}
```

As you can see the code is exactly the same. We are using the uniform to correctly project our coordinates taking into consideration our frustum, position, scale and rotation information. Another important thing to think about is, why don’t we pass the translation, rotation and scale matrices instead of combining them into a world matrix? The reason is that we should try to limit the matrices we use in our shaders. Also keep in mind that the matrix multiplication that we do in our shader is done once per each vertex. The projection matrix does not change between render calls and the world matrix does not change per `Entity` instance. If we passed the translation, rotation and scale matrices independently we would be doing many more matrix multiplications. Think about a model with tons of vertices. That’s a lot of extra operations.

But you may think now that if the model matrix does not change per `Entity` instance, why didn't we do the matrix multiplication in our Java class? We could multiply the projection matrix and the model matrix just once per `Entity` and send it as a single uniform. In this case we would be saving many more operations, right? The answer is that this is a valid point for now, but when we add more features to our game engine we will need to operate with world coordinates in the shaders anyway, so it’s better to handle those two matrices in an independent way.

Finally another very important aspect to remark is the order of multiplication of the matrices. We first need to multiply position information by model matrix, that is we transform model coordinates in world space. After that, we apply the projection. Keep in mind that matrix multiplication is not commutative, so order is very important.

Now we need to modify the `Main` class to adapt to the new way of loading models and entities and the coordinates of indices of a 3D cube.
```java
public class Main implements IAppLogic {

    private Entity cubeEntity;
    private Vector4f displInc = new Vector4f();
    private float rotation;

    public static void main(String[] args) {
        ...
        Engine gameEng = new Engine("chapter-06", new Window.WindowOptions(), main);
        ...
    }
    ...
    @Override
    public void init(Window window, Scene scene, Render render) {
        float[] positions = new float[]{
                // VO
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
        };
        float[] colors = new float[]{
                0.5f, 0.0f, 0.0f,
                0.0f, 0.5f, 0.0f,
                0.0f, 0.0f, 0.5f,
                0.0f, 0.5f, 0.5f,
                0.5f, 0.0f, 0.0f,
                0.0f, 0.5f, 0.0f,
                0.0f, 0.0f, 0.5f,
                0.0f, 0.5f, 0.5f,
        };
        int[] indices = new int[]{
                // Front face
                0, 1, 3, 3, 1, 2,
                // Top Face
                4, 0, 3, 5, 4, 3,
                // Right face
                3, 2, 7, 5, 3, 7,
                // Left face
                6, 1, 0, 6, 0, 4,
                // Bottom face
                2, 1, 6, 2, 6, 7,
                // Back face
                7, 6, 4, 7, 4, 5,
        };
        List<Mesh> meshList = new ArrayList<>();
        Mesh mesh = new Mesh(positions, colors, indices);
        meshList.add(mesh);
        String cubeModelId = "cube-model";
        Model model = new Model(cubeModelId, meshList);
        scene.addModel(model);

        cubeEntity = new Entity("cube-entity", cubeModelId);
        cubeEntity.setPosition(0, 0, -2);
        scene.addEntity(cubeEntity);
    }
    ...
}
```

In order to draw a cube we just need to define eight vertices. Since we have 4 more vertices we need to update the array of colors

![Cube coords](cube_coords.png)

Since a cube is made of six faces we need to draw twelve triangles \(two per face\), so we need to update the indices array. Remember that triangles must be defined in counter-clock wise order. If you do this by hand, is easy to make mistakes. Always put the face that you want to define indices for in front of you. Then, identify the vertices and draw the triangles in counter-clock wise order. Finally, we create a model with just one mesh and an entity associated to that model.

We will first use the `input` method to modify the cube position by using cursor arrows and its scale by using `Z` and `X`key. We just need to detect the key that has been pressed, update the cube entity position and /or scale and finally update its model matrix.
```java
public class Main implements IAppLogic {
    ...
    public void input(Window window, Scene scene, long diffTimeMillis) {
        displInc.zero();
        if (window.isKeyPressed(GLFW_KEY_UP)) {
            displInc.y = 1;
        } else if (window.isKeyPressed(GLFW_KEY_DOWN)) {
            displInc.y = -1;
        }
        if (window.isKeyPressed(GLFW_KEY_LEFT)) {
            displInc.x = -1;
        } else if (window.isKeyPressed(GLFW_KEY_RIGHT)) {
            displInc.x = 1;
        }
        if (window.isKeyPressed(GLFW_KEY_A)) {
            displInc.z = -1;
        } else if (window.isKeyPressed(GLFW_KEY_Q)) {
            displInc.z = 1;
        }
        if (window.isKeyPressed(GLFW_KEY_Z)) {
            displInc.w = -1;
        } else if (window.isKeyPressed(GLFW_KEY_X)) {
            displInc.w = 1;
        }

        displInc.mul(diffTimeMillis / 1000.0f);

        Vector3f entityPos = cubeEntity.getPosition();
        cubeEntity.setPosition(displInc.x + entityPos.x, displInc.y + entityPos.y, displInc.z + entityPos.z);
        cubeEntity.setScale(cubeEntity.getScale() + displInc.w);
        cubeEntity.updateModelMatrix();
    }
    ...
}
```

In order to better view the cube we will change code that rotates the model in the `Main` class to rotate along the three axes. We will do this in the `update` method.
```java
public class Main implements IAppLogic {
    ...
    public void update(Window window, Scene scene, long diffTimeMillis) {
        rotation += 1.5;
        if (rotation > 360) {
            rotation = 0;
        }
        cubeEntity.setRotation(1, 1, 1, (float) Math.toRadians(rotation));
        cubeEntity.updateModelMatrix();
    }
    ...
}
```

And that’s all. We are now able to display a spinning 3D cube. You can now compile and run your example and you will obtain something like this.

![Cube with no depth tests](cube_no_depth_test.png)

There is something weird with this cube. Some faces are not being painted correctly. What is happening? The reason why the cube has this aspect is that the triangles that compose the cube are being drawn in a sort of random order. The pixels that are far away should be drawn before pixels that are closer. This is not happening right now and in order to do that we must enable depth testing.

This can be done in the constructor of the `Render` class:

```java
public class Render {
    ...
    public Render() {
        GL.createCapabilities();
        glEnable(GL_DEPTH_TEST);
        ...
    }
    ...
}
```

Now our cube is being rendered correctly!

![Cube with depth test](cube_depth_test.png)

[Next chapter](../chapter-07/chapter-07.md)