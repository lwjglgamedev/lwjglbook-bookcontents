# Chapter 03 - Your first triangle

In this chapter we will render our first triangle to the screen and introduce the basis of a programmable graphics pipeline. But, prior to that, we will explain first the basis of coordinate systems. Trying to introduce some fundamental mathematical concepts in a simple way to support the techniques and topics that we will address in subsequent chapters. We will assume some simplifications which may sacrifice preciseness for the sake of legibility.

You can find the complete source code for this chapter [here](https://github.com/lwjglgamedev/lwjglbook/tree/main/chapter-03).

## A brief about coordinates

We locate objects in space by specifying their coordinates. Think about a map. You specify a point on a map by stating its latitude or longitude. With just a pair of numbers a point is precisely identified. That pair of numbers are the point coordinates (things are a little bit more complex in reality, since a map is a projection of a non perfect ellipsoid, the earth, so more data is needed but it’s a good analogy).

A coordinate system is a system which employs one or more numbers, that is, one or more components to uniquely specify the position of a point. There are different coordinate systems (Cartesian, polar, etc.) and you can transform coordinates from one system to another. We will use the Cartesian coordinate system.

In the Cartesian coordinate system, for two dimensions, a coordinate is defined by two numbers that measure the signed distance to two perpendicular axes, x and y.

![Cartesian Coordinate System](cartesian_coordinate_system.png) 

Continuing with the map analogy, coordinate systems define an origin. For geographic coordinates the origin is set to the point where the equator and the zero meridian cross. Depending on where we set the origin, coordinates for a specific point are different. A coordinate system may also define the orientation of the axis. In the previous figure, the x coordinate increases as long as we move to the right and the y coordinate increases as we move upwards. But we could also define an alternative Cartesian coordinate system with different axis orientation in which we would obtain different coordinates.
 
![Alternative Cartesian Coordinate System](alt_cartesian_coordinate_system.png)

As you can see we need to define some arbitrary parameters, such as the origin and the axis orientation in order to give the appropriate meaning to the pair of numbers that constitute a coordinate. We will refer to that coordinate system with the set of arbitrary parameters as the coordinate space. In order to work with a set of coordinates we must use the same coordinate space. The good news is that we can transform coordinates from one space to another just by performing translations and rotations.

If we are dealing with 3D coordinates we need an additional axis, the z axis. 3D coordinates will be formed by a set of three numbers (x, y, z). 
 
![3D Cartesian Coordinate System](3d_cartesian_coordinate_system.png)

As in 2D Cartesian coordinate spaces we can change the orientation of the axes in 3D coordinate spaces as long as the axes are perpendicular. The next figure shows another 3D coordinate space.
 
![Alternative 3D Cartesian Coordinate System](alt_3d_cartesian_coordinate_system.png)

3D coordinates can be classified in two types: left handed and right handed. How do you know which type it is? Take your hand and form a “L” between your thumb and your index fingers, the middle finger should point in a direction perpendicular to the other two. The thumb should point to the direction where the x axis increases, the index finger should point where the y axis increases and the middle finger should point where the z axis increases. If you are able to do that with your left hand, then it's left handed, if you need to use your right hand is right-handed.

![Right Handed vs Left Handed](righthanded_lefthanded.png) 

2D coordinate spaces are all equivalent since by applying rotation we can transform from one to another. 3D coordinate spaces, on the contrary, are not all equal. You can only transform from one to another by applying rotation if they both have the same handedness, that is, if both are left handed or right handed.

Now that we have defined some basic topics let’s talk about some commonly used terms when dealing with 3D graphics. When we explain in later chapters how to render 3D models we will see that we use different 3D coordinate spaces, that is because each of those coordinate spaces has a context, a purpose. A set of coordinates is meaningless unless it refers to something. When you examine this coordinates (40.438031, -3.676626) they may say something to you or not. But if I say that they are geometric coordinates (latitude and longitude) you will see that they are the coordinates of a place in Madrid.

When we will load 3D objects we will get a set of 3D coordinates. Those coordinates are expressed in a 3D coordinate space which is called object coordinate space. When the graphics designers are creating those 3D models they don’t know anything about the 3D scene that this model will be displayed in, so they can only define the coordinates using a coordinate space that is only relevant for the model.

When we will be drawing a 3D scene all of our 3D objects will be relative to the so called world space coordinate space. We will need to transform from 3D object space to world space coordinates. Some objects will need to be rotated, stretched or enlarged and translated in order to be displayed properly in a 3D scene.

We will also need to restrict the range of the 3D space that is shown, which is like moving a camera through our 3D space. Then we will need to transform world space coordinates to camera or view space coordinates. Finally these coordinates need to be transformed to screen coordinates, which are 2D, so we need to project 3D view coordinates to a 2D screen coordinate space.

The following picture shows OpenGL coordinates, (the z axis is perpendicular to the screen) and coordinates are between -1 and +1.

![OpenGL coordinates](opengl_coordinates.png)

## Your first triangle

Now we can start learning the processes that takes place while rendering a scene using OpenGL. If you are used to older versions of OpenGL, that is fixed-function pipeline, you may end this chapter wondering why it needs to be so complex. You may end up thinking that drawing a simple shape to the screen should not require so many concepts and lines of code. Let me give you an advice for those of you that think that way. It is actually simpler and much more flexible. You only need to give it a chance. Modern OpenGL lets you think in one problem at a time and it lets you organize your code and processes in a more logical way.

The sequence of steps that ends up drawing a 3D representation into your 2D screen is called the graphics pipeline. First versions of OpenGL employed a model which was called fixed-function pipeline. This model employed a set of steps in the rendering process which defined a fixed set of operations. The programmer was constrained to the set of functions available for each step and could set some parameters to tweak it. Thus, the effects and operations that could be applied were limited by the API itself \(for instance, “set fog” or “add light”, but the implementation of those functions were fixed and could not be changed\).

The graphics pipeline was composed of these steps:

![Graphics Pipeline](rendering_pipeline.png)

OpenGL 2.0 introduced the concept of programmable pipeline. In this model, the different steps that compose the graphics pipeline can be controlled or programmed by using a set of specific programs called shaders. The following picture depicts a simplified version of the OpenGL programmable pipeline:

![Programmable pipeline](rendering_pipeline_2.png)

The rendering starts taking as its input a list of vertices in the form of Vertex Buffers. But, what is a vertex? A vertex is any data structure that can be used as an input to render a scene. By now you can think as a structure that describes a point in 2D or 3D space. And how do you describe a point in a 3D space? By specifying its x, y and z coordinates. And what is a Vertex Buffer? A Vertex Buffer is another data structure that packs all the vertices that need to be rendered, by using vertex arrays, and makes that information available to the shaders in the graphics pipeline.

Those vertices are processed by the vertex shader whose main purpose is to calculate the projected position of each vertex into the screen space. This shader can generate also other outputs related to color or texture, but its main goal is to project the vertices into the screen space, that is, to generate dots.

The geometry processing stage connects the vertices that are transformed by the vertex shader to form triangles. It does so by taking into consideration the order in which the vertices were stored and grouping them using different models. Why triangles? A triangle is like the basic work unit for graphic cards. It’s a simple geometric shape that can be combined and transformed to construct complex 3D scenes. This stage can also use a specific shader to group the vertices.

The rasterization stage takes the triangles generated in the previous stages, clips them and transforms them into pixel-sized fragments.
 Those fragments are used during the fragment processing stage by the fragment shader to generate pixels assigning them the final color that gets written into the framebuffer. The framebuffer is the final result of the graphics pipeline. It holds the value of each pixel that should be drawn to the screen.

Keep in mind that 3D cards are designed to parallelize all the operations described above. The input data is processed in parallel in order to generate the final scene.

So let's start writing our first shader program. Shaders are written by using the GLSL language \(OpenGL Shading Language\) which is based on ANSI C. First we will create a file named “`scene.vert`” \(the extension is for Vertex Shader\) under the `resources\shaders` directory with the following content:

```glsl
#version 330

layout (location=0) in vec3 inPosition;

void main()
{
    gl_Position = vec4(inPosition, 1.0);
}
```

The first line is a directive that states the version of the GLSL language we are using. The following table relates the GLSL version, the OpenGL that matches that version and the directive to use \(Wikipedia: [https://en.wikipedia.org/wiki/OpenGL\_Shading\_Language\#Versions](https://en.wikipedia.org/wiki/OpenGL_Shading_Language#Versions)\).

| GLS Version | OpenGL Version | Shader Preprocessor |
| --- | --- | --- |
| 1.10.59 | 2.0 | \#version 110 |
| 1.20.8 | 2.1 | \#version 120 |
| 1.30.10 | 3.0 | \#version 130 |
| 1.40.08 | 3.1 | \#version 140 |
| 1.50.11 | 3.2 | \#version 150 |
| 3.30.6 | 3.3 | \#version 330 |
| 4.00.9 | 4.0 | \#version 400 |
| 4.10.6 | 4.1 | \#version 410 |
| 4.20.11 | 4.2 | \#version 420 |
| 4.30.8 | 4.3 | \#version 430 |
| 4.40 | 4.4 | \#version 440 |
| 4.50 | 4.5 | \#version 450 |

The second line specifies the input format for this shader. Data in an OpenGL buffer can be whatever we want, that is, the language does not force you to pass a specific data structure with a predefined semantic. From the point of view of the shader it is expecting to receive a buffer with data. It can be a position, a position with some additional information or whatever else we want. In this example, from vertex shader's perspective is just receiving an array of floats. When we fill the buffer, we define the buffer chunks that are going to be processed by the shader.

So, first we need to get that chunk into something that’s meaningful to us. In this case we are saying that, starting from the position 0, we are expecting to receive a vector composed of 3 attributes \(x, y, z\).

The shader has a main block like any other C program which in this case is very simple. It is just returning the received position in the output variable `gl_Position` without applying any transformation. You now may be wondering why the vector of three attributes has been converted into a vector of four attributes \(vec4\). This is because `gl_Position` is expecting the result in vec4 format since it is using homogeneous coordinates. That is, it’s expecting something in the form \(x, y, z, w\), where w represents an extra dimension. Why add another dimension? In later chapters you will see that most of the operations we need to do are based on vectors and matrices. Some of those operations cannot be combined if we do not have that extra dimension. For instance we could not combine rotation and translation operations. \(If you want to learn more on this, this extra dimension allow us to combine affine and linear transformations. You can learn more about this by reading the excellent book “3D Math Primer for Graphics and Game Development", by Fletcher Dunn and Ian Parberry\).

Let us now have a look at our first fragment shader. We will create a file named “`scene.frag`” \(the extension is for Fragment Shader\) under the resources directory with the following content:

```glsl
#version 330

out vec4 fragColor;

void main()
{
    fragColor = vec4(1.0, 0.0, 0.0, 1.0);
}
```

The structure is quite similar to our vertex shader. In this case we will set a fixed color for each fragment. The output variable is defined in the second line and set as a vec4 fragColor.

Now that we have our shaders created, how do we use them? We will need to create a new class named `ShaderProgram` which basically receives the source code of the different shader modules (vertex, fragment) and complies then and links them together to generate a shader program.
This is the sequence of steps we need to follow:  
1.    Create an OpenGL program.  
2.    Load the shader program modules (vertex or fragment shaders).  
3.    For each shader, create a new shader module and specify its type \(vertex, fragment\).  
4.    Compile the shader.  
5.    Attach the shader to the program.  
6.    Link the program.

At the end the shader program will be loaded in the GPU and we can use it by referencing an identifier, the program identifier.
```java
package org.lwjglb.engine.graph;

import org.lwjgl.opengl.GL30;
import org.lwjglb.engine.Utils;

import java.util.*;

import static org.lwjgl.opengl.GL30.*;

public class ShaderProgram {

    private final int programId;

    public ShaderProgram(List<ShaderModuleData> shaderModuleDataList) {
        programId = glCreateProgram();
        if (programId == 0) {
            throw new RuntimeException("Could not create Shader");
        }

        List<Integer> shaderModules = new ArrayList<>();
        shaderModuleDataList.forEach(s -> shaderModules.add(createShader(Utils.readFile(s.shaderFile), s.shaderType)));

        link(shaderModules);
    }

    public void bind() {
        glUseProgram(programId);
    }

    public void cleanup() {
        unbind();
        if (programId != 0) {
            glDeleteProgram(programId);
        }
    }

    protected int createShader(String shaderCode, int shaderType) {
        int shaderId = glCreateShader(shaderType);
        if (shaderId == 0) {
            throw new RuntimeException("Error creating shader. Type: " + shaderType);
        }

        glShaderSource(shaderId, shaderCode);
        glCompileShader(shaderId);

        if (glGetShaderi(shaderId, GL_COMPILE_STATUS) == 0) {
            throw new RuntimeException("Error compiling Shader code: " + glGetShaderInfoLog(shaderId, 1024));
        }

        glAttachShader(programId, shaderId);

        return shaderId;
    }

    public int getProgramId() {
        return programId;
    }

    private void link(List<Integer> shaderModules) {
        glLinkProgram(programId);
        if (glGetProgrami(programId, GL_LINK_STATUS) == 0) {
            throw new RuntimeException("Error linking Shader code: " + glGetProgramInfoLog(programId, 1024));
        }

        shaderModules.forEach(s -> glDetachShader(programId, s));
        shaderModules.forEach(GL30::glDeleteShader);
    }

    public void unbind() {
        glUseProgram(0);
    }

    public void validate() {
        glValidateProgram(programId);
        if (glGetProgrami(programId, GL_VALIDATE_STATUS) == 0) {
            throw new RuntimeException("Error validating Shader code: " + glGetProgramInfoLog(programId, 1024));
        }
    }

    public record ShaderModuleData(String shaderFile, int shaderType) {
    }
}
```

The constructor of the `ShaderProgram` receives a list of `ShaderModuleData` instances which define the shader module type (vertex, fragment, etc.) and the path to the source file which contains the shader module code. The constructor starts by creating a new OpenGL shader program by compiling firs each shader module (by invoking the `createShader` method) and finally linking all together (by invoking the `link` method). Once the shader program has been linked, the compiled vertex and fragment shaders can be freed up \(by calling `glDetachShader`\).

The `validate` method, basically calls the `glValidateProgram` function. This function is used mainly for debugging purposes, and it should not be used when your game reaches production stage. This method tries to validate if the shader is correct given the **current OpenGL state**. This means, that validation may fail in some cases even if the shader is correct, due to the fact that the current state is not complete enough to run the shader \(some data may have not been uploaded yet\). You should call it when all required input and output data is properly bound (better just before performing any drawing call).

`ShaderProgram` also provides methods to use this program for rendering, that is binding it, another one for unbinding (when we are done with it) and finally, a cleanup method to free all the resources when they are no longer needed.

We will create an utility class named `Utils`, which in this case defines a public method to load a file into a `String`:

```java
package org.lwjglb.engine;

import java.io.IOException;
import java.nio.file.*;

public class Utils {

    private Utils() {
        // Utility class
    }

    public static String readFile(String filePath) {
        String str;
        try {
            str = new String(Files.readAllBytes(Paths.get(filePath)));
        } catch (IOException excp) {
            throw new RuntimeException("Error reading file [" + filePath + "]", excp);
        }
        return str;
    }
}
```

We will need also a new class, named `Scene`, which will hold values of our 3D scene, such as models, lights, etc. By now it just tores the meshes (set of vertices) of the models we want to draw. This is the source code for that class:

```java
package org.lwjglb.engine.scene;

import org.lwjglb.engine.graph.Mesh;

import java.util.*;

public class Scene {

    private Map<String, Mesh> meshMap;

    public Scene() {
        meshMap = new HashMap<>();
    }

    public void addMesh(String meshId, Mesh mesh) {
        meshMap.put(meshId, mesh);
    }

    public void cleanup() {
        meshMap.values().forEach(Mesh::cleanup);
    }

    public Map<String, Mesh> getMeshMap() {
        return meshMap;
    }
}
```

As you can see it just stores `Mesh` instances in a Map, which is later on used for drawing. But what is a `Mesh`? It is basically our way to load vertices data into the GPU so it can be used for render. Prior to describe in detail the `Mesh` class, let's see how it can be used in our `Main` class:
```java
public class Main implements IAppLogic {

    public static void main(String[] args) {
        Main main = new Main();
        Engine gameEng = new Engine("chapter-03", new Window.WindowOptions(), main);
        gameEng.start();
    }
    ...
    @Override
    public void init(Window window, Scene scene, Render render) {
        float[] positions = new float[]{
                0.0f, 0.5f, 0.0f,
                -0.5f, -0.5f, 0.0f,
                0.5f, -0.5f, 0.0f
        };
        Mesh mesh = new Mesh(positions, 3);
        scene.addMesh("triangle", mesh);
    }
    ...
}
```

In the `init` method, we define an array of floats that contain the coordinates of the vertices of a triangle. As you can see there’s no structure in that array, we just dump there all the coordinates. As it is right now, OpenGL cannot know the structure of that data. It’s just a sequence of floats. The following picture depicts the triangle in our coordinate system.

![Triangle](triangle_coordinates.png)

The class that defines the structure of that data and loads it in the GPU is the `Mesh` class which is defined like this:
```java
package org.lwjglb.engine.graph;

import org.lwjgl.opengl.GL30;
import org.lwjgl.system.MemoryStack;

import java.nio.FloatBuffer;
import java.util.*;

import static org.lwjgl.opengl.GL30.*;

public class Mesh {

    private int numVertices;
    private int vaoId;
    private List<Integer> vboIdList;

    public Mesh(float[] positions, int numVertices) {
        this.numVertices = numVertices;
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

        glBindBuffer(GL_ARRAY_BUFFER, 0);
        glBindVertexArray(0);

        MemoryUtil.memFree(positionsBuffer);
    }

    public void cleanup() {
        vboIdList.forEach(GL30::glDeleteBuffers);
        glDeleteVertexArrays(vaoId);
    }

    public int getNumVertices() {
        return numVertices;
    }

    public final int getVaoId() {
        return vaoId;
    }
}
```

We are introducing now two important concepts, Vertex Array Objects \(VAOs\) and Vertex Buffer Object \(VBOs\). If you get lost in the code above remember that at the end what we are doing is sending the data that models the objects we want to draw to the graphics card memory. When we store it we get an identifier that serves us later to refer to it while drawing.

Let us first start with Vertex Buffer Object \(VBOs\). A VBO is just a memory buffer stored in the graphics card memory that stores vertices. This is where we will transfer our array of floats that model a triangle. As we said before, OpenGL does not know anything about our data structure. In fact it can hold not just coordinates but other information, such as textures, color, etc. A Vertex Array Objects \(VAOs\) is an object that contains one or more VBOs which are usually called attribute lists. Each attribute list can hold one type of data: position, color, texture, etc. You are free to store whichever you want in each slot.

A VAO is like a wrapper that groups a set of definitions for the data that is going to be stored in the graphics card. When we create a VAO we get an identifier. We use that identifier to render it and the elements it contains using the definitions we specified during its creation.

So let us review the code above. The first thing that we do is to create the VAO (by calling the `glGenVertexArrays` function) and bind it (by calling the `glBindVertexArray` function). After that,  we need to create the VBO (by calling the `glGenBuffers`) and put the data into it. In order to do so, we store our array of floats into a `FloatBuffer`. This is mainly due to the fact that we must interface with the OpenGL library, which is C-based, so we must transform our array of floats into something that can be managed by the library.

We use the `MemoryUtil` class to create the buffer in off-heap memory so that it's accessible by the OpenGL library. We have store the data \(with the put method\). Remember, that Java objects, are allocated in a space called the heap. The heap is a large bunch of memory reserved in the JVM's process memory. Memory stored in the heap cannot be accessed by native code \(JNI, the mechanism that allows calling native code from Java does not allow that\).  The only way of sharing memory data between Java and native code is by directly allocating memory in Java.

If you come from previous versions of LWJGL it's important to stress out a few topics. You may have noticed that we do not use the utility class `BufferUtils` to create the buffers. Instead we use the `MemoryUtil` class. This is due to the fact that `BufferUtils` was not very efficient, and has been maintained only for backwards compatibility. Instead, LWJGL 3 proposes two methods for buffer management:

* Auto-managed buffers, that is, buffers that are automatically collected by the Garbage Collector. These buffers are mainly used for short-lived operations, or for data that is transferred to the GPU and does not need to be present in the process memory. This is achieved by using the `org.lwjgl.system.MemoryStack` class.
* Manually managed buffers. In this case we need to carefully free them once we are finished. These buffers are intended for long time operations or for large amounts of data. This is achieved by using the `MemoryUtil` class.

You can consult the details here:  [https://blog.lwjgl.org/memory-management-in-lwjgl-3/](https://blog.lwjgl.org/memory-management-in-lwjgl-3/ "here").

In this specific case, positions data is short lived, once we have loaded the data, we are done with that buffer. You may think, we then you do not use the `org.lwjgl.system.MemoryStack` class? The reason to use the second approach (`MemoryUtil` class) is that LWJGL's stack is limited. If you end up loading large modes you may consume all the available space and get and "Out of stack space" exception. The drawback for this approach is that we need to manually free the memory once we are done with it by calling `MemoryUtil.memFree`.

After that, we bind the VBO (by calling the `glBindBuffer`) and load the data into it (by calling the `glBufferData` function). Now comes the most important part. We need to define the structure of our data and store it in one of the attribute lists of the VAO. This is done with the following line.

```java
glVertexAttribPointer(0, 3, GL_FLOAT, false, 0, 0);
```

The parameters are:

* index: Specifies the location where the shader expects this data.
* size: Specifies the number of components per vertex attribute \(from 1 to 4\). In this case, we are passing 3D coordinates, so it should be 3.
* type: Specifies the type of each component in the array, in this case a float.
* normalized: Specifies if the values should be normalized or not.
* stride: Specifies the byte offset between consecutive generic vertex attributes. \(We will explain it later\).
* offset: Specifies an offset to the first component in the buffer.

After we are finished with our VBO and VAO so we can unbind them \(bind them to 0\)

Since  we are using auto-managed buffers, once the `try` / `catch` block finished, the buffer is automatically cleaned up.

The `Mesh` class is completed by the `cleanup` method which basically frees the VAO and the VBO and some getter methods to get the number of vertices of the mesh and the id of the VAO. When rendering this elements, we will use the VAO id when using drawing operations.


Now let's put all of this into work. We will create a new class named `SceneRender` which will perform the render of all the models in our scene and is defined like this:
```java
package org.lwjglb.engine.graph;

import org.lwjglb.engine.Window;
import org.lwjglb.engine.scene.Scene;

import java.util.*;

import static org.lwjgl.opengl.GL30.*;

public class SceneRender {

    private ShaderProgram shaderProgram;

    public SceneRender() {
        List<ShaderProgram.ShaderModuleData> shaderModuleDataList = new ArrayList<>();
        shaderModuleDataList.add(new ShaderProgram.ShaderModuleData("resources/shaders/scene.vert", GL_VERTEX_SHADER));
        shaderModuleDataList.add(new ShaderProgram.ShaderModuleData("resources/shaders/scene.frag", GL_FRAGMENT_SHADER));
        shaderProgram = new ShaderProgram(shaderModuleDataList);
    }

    public void cleanup() {
        shaderProgram.cleanup();
    }

    public void render(Scene scene) {
        shaderProgram.bind();

        scene.getMeshMap().values().forEach(mesh -> {
                    glBindVertexArray(mesh.getVaoId());
                    glDrawArrays(GL_TRIANGLES, 0, mesh.getNumVertices());
                }
        );

        glBindVertexArray(0);

        shaderProgram.unbind();
    }
}
```

As you can see, in the constructor, we create two `ShaderModuleData` instances (one for vertex shader and the other one for fragment) and creates a shader program. We define a `cleanup` method to free the resources (in this case the shader program) and a `render` method which is the one which performs the drawing. This method starts by using the shader program by calling its `bind` method. Then, we iterate over the meshes stored in the `Scene` instance, bind them (by calling the `glBindVertexArray` function) and draw the vertices of the VAO (by calling the `glDrawArrays` function). Finally, we unbind the VAO and the shader program to restore the state.

Finally, we just need to update the `Render` class to use the `SceneRender` class.
```java
package org.lwjglb.engine.graph;

import org.lwjgl.opengl.GL;
import org.lwjglb.engine.Window;
import org.lwjglb.engine.scene.Scene;

public class Render {

    private SceneRender sceneRender;

    public Render() {
        GL.createCapabilities();
        sceneRender = new SceneRender();
    }

    public void cleanup() {
        sceneRender.cleanup();
    }

    public void render(Window window, Scene scene) {
        ...
        glViewport(0, 0, window.getWidth(), window.getHeight());
        sceneRender.render(scene);
    }
}
```

 The `render` method starts by clearing the framebuffer and setting the view port (by calling the `glViewport` method) to the window dimensions. That is, we set the rendering area to those dimensions (This does not need to be done for every frame, but if we want to support window resizing we can do it this way to adapt to potential changes in each frame). After that we just invoke the `render` method over the `SceneRender` instance. And, that’s all! If you followed the steps carefully you will see something like this:

![Triangle game](triangle_window.png)

Our first triangle! You may think that this will not make it into the top ten game list, and you will be totally right. You may also think that this has been too much work for drawing a boring triangle. But keep in mind that we are introducing key concepts and preparing the base infrastructure to do more complex things. Please be patient and continue reading.

[Next chapter](../chapter-04/chapter-04.md)
