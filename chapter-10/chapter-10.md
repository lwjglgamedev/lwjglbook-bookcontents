# Chapter 10 - GUI (Imgui)

[Dear ImGui](https://github.com/ocornut/imgui) is a user interface library which can use several backends such as OpenGL and Vulkan. We will use it to display gui controls or to develop HUDs. It provides multiple widgets and the look and fell is easily customizable.

You can find the complete source code for this chapter [here](https://github.com/lwjglgamedev/lwjglbook/tree/main/chapter-10).

## Imgui integration

The first thing is adding Java Imgui wrapper maven dependencies to the project pom.xml. We need to add compile time and runtime dependencies.

```xml
<dependency>
   <groupId>io.github.spair</groupId>
   <artifactId>imgui-java-binding</artifactId>
   <version>${imgui-java.version}</version>
</dependency>
<dependency>
    <groupId>io.github.spair</groupId>
    <artifactId>imgui-java-${native.target}</artifactId>
    <version>${imgui-java.version}</version>
    <scope>runtime</scope>
</dependency>
```

With Imgui we can render windows, panels, etc. like we render any other 3D model, but using only 2D shapes. We set the controls that we want to use and Imgui translates that to a set of vertex buffers that we can render using shaders. This is why it can be used with any backend.

For each vertex, Imgui defines its coordinates (2D coordinates), texture coordinates and the associated color. Therefore, we need to create a new class to model Gui meshes and to create the associated VAO and VBO. The class, named `GuiMesh` is defined like this.

```java
package org.lwjglb.engine.graph;

import imgui.ImDrawData;

import static org.lwjgl.opengl.GL15.*;
import static org.lwjgl.opengl.GL20.*;
import static org.lwjgl.opengl.GL30.*;

public class GuiMesh {

    private int indicesVBO;
    private int vaoId;
    private int verticesVBO;

    public GuiMesh() {
        vaoId = glGenVertexArrays();
        glBindVertexArray(vaoId);

        // Single VBO
        verticesVBO = glGenBuffers();
        glBindBuffer(GL_ARRAY_BUFFER, verticesVBO);
        glEnableVertexAttribArray(0);
        glVertexAttribPointer(0, 2, GL_FLOAT, false, ImDrawData.sizeOfImDrawVert(), 0);
        glEnableVertexAttribArray(1);
        glVertexAttribPointer(1, 2, GL_FLOAT, false, ImDrawData.sizeOfImDrawVert(), 8);
        glEnableVertexAttribArray(2);
        glVertexAttribPointer(2, 4, GL_UNSIGNED_BYTE, true, ImDrawData.sizeOfImDrawVert(), 16);

        indicesVBO = glGenBuffers();

        glBindBuffer(GL_ARRAY_BUFFER, 0);
        glBindVertexArray(0);
    }

    public void cleanup() {
        glDeleteBuffers(indicesVBO);
        glDeleteBuffers(verticesVBO);
        glDeleteVertexArrays(vaoId);
    }

    public int getIndicesVBO() {
        return indicesVBO;
    }

    public int getVaoId() {
        return vaoId;
    }

    public int getVerticesVBO() {
        return verticesVBO;
    }
}
```

As you can see, we use a single VBO but we define several attributes for the positions, texture coordinates and color. In this case, we do not populate the buffers with data, we will see later on how we will use it.

We need also to let the application create GUI controls and react to the user input. In order to support this, we will define a new interface named `IGuiInstance` which is defined like this:

```java
package org.lwjglb.engine;

import org.lwjglb.engine.scene.Scene;

public interface IGuiInstance {
    void drawGui();

    boolean handleGuiInput(Scene scene, Window window);
}
```

The method `drawGui` will be used to construct the GUI, this where we will define the window and widgets that will be used to construct the GUI meshes. We will use the `handleGuiInput` method to process input events in the GUI. It returns a boolean value to state that the input has been processed by the GUI or not. For example, if we display an overlapping window we may not be interested in keep processing keystrokes in the game logic. You can use the return value to control that. We will store the specific implementation of `IGuiInstance` interface in the `Scene` class.

```java
public class Scene {
    ...
    private IGuiInstance guiInstance;
    ...
    public IGuiInstance getGuiInstance() {
        return guiInstance;
    }
    ...
    public void setGuiInstance(IGuiInstance guiInstance) {
        this.guiInstance = guiInstance;
    }
}
```

The next step will be to create a new class to render our GUI, which will be named `GuiRender` and starts like this:

```java
package org.lwjglb.engine.graph;

import imgui.*;
import imgui.type.ImInt;
import org.joml.Vector2f;
import org.lwjglb.engine.*;
import org.lwjglb.engine.scene.Scene;

import java.nio.ByteBuffer;
import java.util.*;

import static org.lwjgl.opengl.GL32.*;
import static org.lwjgl.glfw.GLFW.*;

public class GuiRender {

    private GuiMesh guiMesh;
    private GLFWKeyCallback prevKeyCallBack;
    private Vector2f scale;
    private ShaderProgram shaderProgram;
    private Texture texture;
    private UniformsMap uniformsMap;

    public GuiRender(Window window) {
        List<ShaderProgram.ShaderModuleData> shaderModuleDataList = new ArrayList<>();
        shaderModuleDataList.add(new ShaderProgram.ShaderModuleData("resources/shaders/gui.vert", GL_VERTEX_SHADER));
        shaderModuleDataList.add(new ShaderProgram.ShaderModuleData("resources/shaders/gui.frag", GL_FRAGMENT_SHADER));
        shaderProgram = new ShaderProgram(shaderModuleDataList);
        createUniforms();
        createUIResources(window);
        setupKeyCallBack(window);
    }

    public void cleanup() {
        shaderProgram.cleanup();
        texture.cleanup();
        if (prevKeyCallBack != null) {
            prevKeyCallBack.free();
        }
    }
    ...
}
```

As you can see, most of the stuff here will be very familiar to you, we just set up the shaders and the uniforms. Since we will need to set up a custom key callback to handle ImGui input text controls, we need to keep track of a previous key callback in `prevKeyCallBack` to properly use it and free it. In addition to that, there is a new method called `createUIResources` which is defined like this:

```java
public class GuiRender {
    ...
    private void createUIResources(Window window) {
        ImGui.createContext();

        ImGuiIO imGuiIO = ImGui.getIO();
        imGuiIO.setIniFilename(null);
        imGuiIO.setDisplaySize(window.getWidth(), window.getHeight());

        ImFontAtlas fontAtlas = ImGui.getIO().getFonts();
        ImInt width = new ImInt();
        ImInt height = new ImInt();
        ByteBuffer buf = fontAtlas.getTexDataAsRGBA32(width, height);
        texture = new Texture(width.get(), height.get(), buf);

        guiMesh = new GuiMesh();
    }
    ...
}
```

In the method above is where we setup Imgui, we first create a context (required to perform any operation), and set up the display size to the window size. Imgui stores the status in an ini file, since we do not want the status to persist between runs we need to set it to null. The next step is to initialize the font atlas and set up a texture which will be used in the shaders so we can render properly texts, etc. The final step is to create the `GuiMesh` instance.

The `createUniforms` just creates a single two float for the scale (we will see later on how it will be used).

```java
public class GuiRender {
    ...
    private void createUniforms() {
        uniformsMap = new UniformsMap(shaderProgram.getProgramId());
        uniformsMap.createUniform("scale");
        scale = new Vector2f();
    }
    ...
}
```

The `setupKeyCallBack` method is required to properly process key events in Imgui and is defined like this:

```java
public class GuiRender {
    ...
    private void setupKeyCallBack(Window window) {
        prevKeyCallBack = glfwSetKeyCallback(window.getWindowHandle(), (handle, key, scancode, action, mods) -> {
            window.keyCallBack(key, action);
            ImGuiIO io = ImGui.getIO();
            if (!io.getWantCaptureKeyboard()) {
                return;
            }
            if (action == GLFW_PRESS) {
                io.addKeyEvent(getImKey(key), true);
            } else if (action == GLFW_RELEASE) {
                io.addKeyEvent(getImKey(key), false);
            }
        }
        );

        glfwSetCharCallback(window.getWindowHandle(), (handle, c) -> {
            ImGuiIO io = ImGui.getIO();
            if (!io.getWantCaptureKeyboard()) {
                return;
            }
            io.addInputCharacter(c);
        });
    }

    private static int getImKey(int key) {
        return switch (key) {
            case GLFW_KEY_TAB -> ImGuiKey.Tab;
            case GLFW_KEY_LEFT -> ImGuiKey.LeftArrow;
            case GLFW_KEY_RIGHT -> ImGuiKey.RightArrow;
            case GLFW_KEY_UP -> ImGuiKey.UpArrow;
            case GLFW_KEY_DOWN -> ImGuiKey.DownArrow;
            case GLFW_KEY_PAGE_UP -> ImGuiKey.PageUp;
            case GLFW_KEY_PAGE_DOWN -> ImGuiKey.PageDown;
            case GLFW_KEY_HOME -> ImGuiKey.Home;
            case GLFW_KEY_END -> ImGuiKey.End;
            case GLFW_KEY_INSERT -> ImGuiKey.Insert;
            case GLFW_KEY_DELETE -> ImGuiKey.Delete;
            case GLFW_KEY_BACKSPACE -> ImGuiKey.Backspace;
            case GLFW_KEY_SPACE -> ImGuiKey.Space;
            case GLFW_KEY_ENTER -> ImGuiKey.Enter;
            case GLFW_KEY_ESCAPE -> ImGuiKey.Escape;
            case GLFW_KEY_APOSTROPHE -> ImGuiKey.Apostrophe;
            case GLFW_KEY_COMMA -> ImGuiKey.Comma;
            case GLFW_KEY_MINUS -> ImGuiKey.Minus;
            case GLFW_KEY_PERIOD -> ImGuiKey.Period;
            case GLFW_KEY_SLASH -> ImGuiKey.Slash;
            case GLFW_KEY_SEMICOLON -> ImGuiKey.Semicolon;
            case GLFW_KEY_EQUAL -> ImGuiKey.Equal;
            case GLFW_KEY_LEFT_BRACKET -> ImGuiKey.LeftBracket;
            case GLFW_KEY_BACKSLASH -> ImGuiKey.Backslash;
            case GLFW_KEY_RIGHT_BRACKET -> ImGuiKey.RightBracket;
            case GLFW_KEY_GRAVE_ACCENT -> ImGuiKey.GraveAccent;
            case GLFW_KEY_CAPS_LOCK -> ImGuiKey.CapsLock;
            case GLFW_KEY_SCROLL_LOCK -> ImGuiKey.ScrollLock;
            case GLFW_KEY_NUM_LOCK -> ImGuiKey.NumLock;
            case GLFW_KEY_PRINT_SCREEN -> ImGuiKey.PrintScreen;
            case GLFW_KEY_PAUSE -> ImGuiKey.Pause;
            case GLFW_KEY_KP_0 -> ImGuiKey.Keypad0;
            case GLFW_KEY_KP_1 -> ImGuiKey.Keypad1;
            case GLFW_KEY_KP_2 -> ImGuiKey.Keypad2;
            case GLFW_KEY_KP_3 -> ImGuiKey.Keypad3;
            case GLFW_KEY_KP_4 -> ImGuiKey.Keypad4;
            case GLFW_KEY_KP_5 -> ImGuiKey.Keypad5;
            case GLFW_KEY_KP_6 -> ImGuiKey.Keypad6;
            case GLFW_KEY_KP_7 -> ImGuiKey.Keypad7;
            case GLFW_KEY_KP_8 -> ImGuiKey.Keypad8;
            case GLFW_KEY_KP_9 -> ImGuiKey.Keypad9;
            case GLFW_KEY_KP_DECIMAL -> ImGuiKey.KeypadDecimal;
            case GLFW_KEY_KP_DIVIDE -> ImGuiKey.KeypadDivide;
            case GLFW_KEY_KP_MULTIPLY -> ImGuiKey.KeypadMultiply;
            case GLFW_KEY_KP_SUBTRACT -> ImGuiKey.KeypadSubtract;
            case GLFW_KEY_KP_ADD -> ImGuiKey.KeypadAdd;
            case GLFW_KEY_KP_ENTER -> ImGuiKey.KeypadEnter;
            case GLFW_KEY_KP_EQUAL -> ImGuiKey.KeypadEqual;
            case GLFW_KEY_LEFT_SHIFT -> ImGuiKey.LeftShift;
            case GLFW_KEY_LEFT_CONTROL -> ImGuiKey.LeftCtrl;
            case GLFW_KEY_LEFT_ALT -> ImGuiKey.LeftAlt;
            case GLFW_KEY_LEFT_SUPER -> ImGuiKey.LeftSuper;
            case GLFW_KEY_RIGHT_SHIFT -> ImGuiKey.RightShift;
            case GLFW_KEY_RIGHT_CONTROL -> ImGuiKey.RightCtrl;
            case GLFW_KEY_RIGHT_ALT -> ImGuiKey.RightAlt;
            case GLFW_KEY_RIGHT_SUPER -> ImGuiKey.RightSuper;
            case GLFW_KEY_MENU -> ImGuiKey.Menu;
            case GLFW_KEY_0 -> ImGuiKey._0;
            case GLFW_KEY_1 -> ImGuiKey._1;
            case GLFW_KEY_2 -> ImGuiKey._2;
            case GLFW_KEY_3 -> ImGuiKey._3;
            case GLFW_KEY_4 -> ImGuiKey._4;
            case GLFW_KEY_5 -> ImGuiKey._5;
            case GLFW_KEY_6 -> ImGuiKey._6;
            case GLFW_KEY_7 -> ImGuiKey._7;
            case GLFW_KEY_8 -> ImGuiKey._8;
            case GLFW_KEY_9 -> ImGuiKey._9;
            case GLFW_KEY_A -> ImGuiKey.A;
            case GLFW_KEY_B -> ImGuiKey.B;
            case GLFW_KEY_C -> ImGuiKey.C;
            case GLFW_KEY_D -> ImGuiKey.D;
            case GLFW_KEY_E -> ImGuiKey.E;
            case GLFW_KEY_F -> ImGuiKey.F;
            case GLFW_KEY_G -> ImGuiKey.G;
            case GLFW_KEY_H -> ImGuiKey.H;
            case GLFW_KEY_I -> ImGuiKey.I;
            case GLFW_KEY_J -> ImGuiKey.J;
            case GLFW_KEY_K -> ImGuiKey.K;
            case GLFW_KEY_L -> ImGuiKey.L;
            case GLFW_KEY_M -> ImGuiKey.M;
            case GLFW_KEY_N -> ImGuiKey.N;
            case GLFW_KEY_O -> ImGuiKey.O;
            case GLFW_KEY_P -> ImGuiKey.P;
            case GLFW_KEY_Q -> ImGuiKey.Q;
            case GLFW_KEY_R -> ImGuiKey.R;
            case GLFW_KEY_S -> ImGuiKey.S;
            case GLFW_KEY_T -> ImGuiKey.T;
            case GLFW_KEY_U -> ImGuiKey.U;
            case GLFW_KEY_V -> ImGuiKey.V;
            case GLFW_KEY_W -> ImGuiKey.W;
            case GLFW_KEY_X -> ImGuiKey.X;
            case GLFW_KEY_Y -> ImGuiKey.Y;
            case GLFW_KEY_Z -> ImGuiKey.Z;
            case GLFW_KEY_F1 -> ImGuiKey.F1;
            case GLFW_KEY_F2 -> ImGuiKey.F2;
            case GLFW_KEY_F3 -> ImGuiKey.F3;
            case GLFW_KEY_F4 -> ImGuiKey.F4;
            case GLFW_KEY_F5 -> ImGuiKey.F5;
            case GLFW_KEY_F6 -> ImGuiKey.F6;
            case GLFW_KEY_F7 -> ImGuiKey.F7;
            case GLFW_KEY_F8 -> ImGuiKey.F8;
            case GLFW_KEY_F9 -> ImGuiKey.F9;
            case GLFW_KEY_F10 -> ImGuiKey.F10;
            case GLFW_KEY_F11 -> ImGuiKey.F11;
            case GLFW_KEY_F12 -> ImGuiKey.F12;
            default -> ImGuiKey.None;
        };
    }
    ...
}
```

First we need to setup a GLFW key callback which first calls `Window` key call back to handle key events and translate GFLW key code sto Imgui ones. When setting a callback we obtain a reference to a previously established one so we can chain them. In this case we will invoke it if the key event is not handled by ImGui. We are not using char callbacks in other parts of the code, but if you do, remember to apply that chain schema also. After that, we set up the state of Imgui according to key pressed or released events. Finally, we need to setup a char call back so text input widgets can process those events.

Let's view the `render` method now:

```java
public class GuiRender {
    ...
    public void render(Scene scene) {
        IGuiInstance guiInstance = scene.getGuiInstance();
        if (guiInstance == null) {
            return;
        }
        guiInstance.drawGui();

        shaderProgram.bind();

        glEnable(GL_BLEND);
        glBlendEquation(GL_FUNC_ADD);
        glBlendFuncSeparate(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA, GL_ONE, GL_ONE_MINUS_SRC_ALPHA);
        glDisable(GL_DEPTH_TEST);
        glDisable(GL_CULL_FACE);

        glBindVertexArray(guiMesh.getVaoId());

        glBindBuffer(GL_ARRAY_BUFFER, guiMesh.getVerticesVBO());
        glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, guiMesh.getIndicesVBO());

        ImGuiIO io = ImGui.getIO();
        scale.x = 2.0f / io.getDisplaySizeX();
        scale.y = -2.0f / io.getDisplaySizeY();
        uniformsMap.setUniform("scale", scale);

        ImDrawData drawData = ImGui.getDrawData();
        int numLists = drawData.getCmdListsCount();
        for (int i = 0; i < numLists; i++) {
            glBufferData(GL_ARRAY_BUFFER, drawData.getCmdListVtxBufferData(i), GL_STREAM_DRAW);
            glBufferData(GL_ELEMENT_ARRAY_BUFFER, drawData.getCmdListIdxBufferData(i), GL_STREAM_DRAW);

            int numCmds = drawData.getCmdListCmdBufferSize(i);
            for (int j = 0; j < numCmds; j++) {
                final int elemCount = drawData.getCmdListCmdBufferElemCount(i, j);
                final int idxBufferOffset = drawData.getCmdListCmdBufferIdxOffset(i, j);
                final int indices = idxBufferOffset * ImDrawData.sizeOfImDrawIdx();

                texture.bind();
                glDrawElements(GL_TRIANGLES, elemCount, GL_UNSIGNED_SHORT, indices);
            }
        }

        glEnable(GL_DEPTH_TEST);
        glEnable(GL_CULL_FACE);
        glDisable(GL_BLEND);
    }
    ...
}
```

The first thing that we do is to check if we have set up an implementation of the `IGuiInstance` interface. If there is no instance, we just return, there is no need to render anything. After that we call the `drawGui` method. That is, in each render call we invoke that method so the Imgui can update its status to be able to generate the proper vertex data. After binding the shader we first enable blending which will allow us to use transparencies. Just by enabling blending, transparencies still will not show up. We need also to instruct OpenGL about how the blending will be applied. This is done through the `glBlendFunc` function. You can check an excellent explanation about the details of the different functions that can be applied [here](https://learnopengl.com/Advanced-OpenGL/Blending).

After that, we need to disable depth testing and face culling for Imgui to work properly. Then, we bind the gui mesh which defines the structure of the data and bind the data and indices buffers. Imgui uses screen coordinates to generate the vertices data, that is `x` values cover the `[0, screen width]` range and `y` values cover the `[0, screen height]`. We will use the `scale` uniform to map from that coordinate system to the `[-1, 1]` range of OpenGL's clip space.

After that, we retrieve the data generated by Imgui to render the GUI. Imgui first organizes the data in what they call command lists. Each command list has a buffer where it stores the vertex and indices data, so we first dump data to the GPU by calling the `glBufferData`. Each command list defines also a set of commands which we will use to generate the draw calls. Each command stores the number of elements to be drawn and the offset to be applied to the buffer in the command list. When we have drawn all the elements we can re-enable the depth test.

Finally, we need to add a `resize` method which will be called any time the window is resized to adjust Imgui display size.

```java
public class GuiRender {
    ...
    public void resize(int width, int height) {
        ImGuiIO imGuiIO = ImGui.getIO();
        imGuiIO.setDisplaySize(width, height);
    }
}
```

We need to update the `UniformsMap` class to add support for 2D vectors:

```java
public class UniformsMap {
    ...
    public void setUniform(String uniformName, Vector2f value) {
        glUniform2f(getUniformLocation(uniformName), value.x, value.y);
    }
}
```

The vertex shader used for rendering the GUI is quite simple (`gui.vert`), we just transform the coordinates so they are in the `[-1, 1]` range and output the texture coordinates and color so they can be used in the fragment shader:

```glsl
#version 330

layout (location=0) in vec2 inPos;
layout (location=1) in vec2 inTextCoords;
layout (location=2) in vec4 inColor;

out vec2 frgTextCoords;
out vec4 frgColor;

uniform vec2 scale;

void main()
{
    frgTextCoords = inTextCoords;
    frgColor = inColor;
    gl_Position = vec4(inPos * scale + vec2(-1.0, 1.0), 0.0, 1.0);
}
```

In the fragment shader (`gui.frag`) we just output the combination of the vertex color and the texture color associated to its texture coordinates:

```glsl
#version 330

in vec2 frgTextCoords;
in vec4 frgColor;

uniform sampler2D txtSampler;

out vec4 outColor;

void main()
{
    outColor = frgColor  * texture(txtSampler, frgTextCoords);
}
```

## Putting it up all together

Now we need to glue all the previous pices to render the GUI. We will first start by using the new `GuiRender` class into the `Render` one.

```java
public class Render {
    ...
    private GuiRender guiRender;
    ...
    public Render(Window window) {
        ...
        guiRender = new GuiRender(window);
    }

    public void cleanup() {
        ...
        guiRender.cleanup();
    }

    public void render(Window window, Scene scene) {
        ...
        guiRender.render(scene);
    }

    public void resize(int width, int height) {
        guiRender.resize(width, height);
    }
}
```

We also need to modify the `Engine` class to include `IGuiInstance` in the update loop and to use its return value to indicate if input has been consumed or not.

```java
public class Engine {
    ...
    public Engine(String windowTitle, Window.WindowOptions opts, IAppLogic appLogic) {
        ...
        render = new Render(window);
        ...
    }
    ...
    private void resize() {
        int width = window.getWidth();
        int height = window.getHeight();
        scene.resize(width, height);
        render.resize(width, height);
    }

    private void run() {
        ...
        IGuiInstance iGuiInstance = scene.getGuiInstance();
        while (running && !window.windowShouldClose()) {
            ...
            if (targetFps <= 0 || deltaFps >= 1) {
                window.getMouseInput().input();
                boolean inputConsumed = iGuiInstance != null && iGuiInstance.handleGuiInput(scene, window);
                appLogic.input(window, scene, now - initialTime, inputConsumed);
            }
            ...
        }
        ...
    }
    ...
}
```

We need also to update the `IAppLogic` interface to use the input consumed return value.

```java
public interface IAppLogic {
    ...
    void input(Window window, Scene scene, long diffTimeMillis, boolean inputConsumed);
    ...
}
```

And finally, we will implement the `IGuiInstance` in the `Main` class:

```java
public class Main implements IAppLogic, IGuiInstance {
    ...
    public static void main(String[] args) {
        ...
        Engine gameEng = new Engine("chapter-10", new Window.WindowOptions(), main);
        ...
    }

    ...
    @Override
    public void drawGui() {
        ImGui.newFrame();
        ImGui.setNextWindowPos(0, 0, ImGuiCond.Always);
        ImGui.showDemoWindow();
        ImGui.endFrame();
        ImGui.render();
    }

    @Override
    public boolean handleGuiInput(Scene scene, Window window) {
        ImGuiIO imGuiIO = ImGui.getIO();
        MouseInput mouseInput = window.getMouseInput();
        Vector2f mousePos = mouseInput.getCurrentPos();
        imGuiIO.addMousePosEvent(mousePos.x, mousePos.y);
        imGuiIO.addMouseButtonEvent(0, mouseInput.isLeftButtonPressed());
        imGuiIO.addMouseButtonEvent(1, mouseInput.isRightButtonPressed());

        return imGuiIO.getWantCaptureMouse() || imGuiIO.getWantCaptureKeyboard();
    }
    ...
    public void input(Window window, Scene scene, long diffTimeMillis, boolean inputConsumed) {
        if (inputConsumed) {
            return;
        }
        ...
    }
}
```

In the `drawGui` method we just setup a new frame, the window position and just invoke the `showDemoWindow` to generate Imgui's demo window. After ending the frame it is very important to call the `render` this is what will generate the set of commands upon the GUI structure defined previously. The `handleGuiInput` first gets mouse position and updates Imgui's IO class with that information and mouse button status. We also return a boolean that indicates that input has been capture by Imgui. Finally, we just need to update the `input` method to receive that flag. In this specific case, if input has already been consumed by the Gui, we just return.

With all those changes you will be able to see Imgui demo window overlapping the rotating cube. You can interact with the different methods and panels to get a glimpse of the capabilities of Imgui.

![Demo window](../.gitbook/assets/demo.png)

[Next chapter](../chapter-11/chapter-11.md)
