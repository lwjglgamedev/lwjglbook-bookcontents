# Chapter 02 - The Game Loop

In this chapter we will start developing our game engine by creating the game loop. The game loop is the core component of every game. It is basically an endless loop which is responsible for periodically handling user input, updating game state and rendering to the screen.

You can find the complete source code for this chapter [here](https://github.com/lwjglgamedev/lwjglbook/chapter-02).

## The basis

The following snippet shows the structure of a game loop:

```java
while (keepOnRunning) {
    input();
    update();
    render();
}
```

The `input` method is responsible of handling user input (key strokes, mouse movements, etc.). The `update` method is responsible of updating game state (enemy positions, AI, etc..) and, finally, theSo, is that all? Are we finished with game loops? Well, not yet. The above snippet has many pitfalls. First of all the speed that the game loop runs at will be different depending on the machine it runs on. If the machine is fast enough the user will not even be able to see what is happening in the game. Moreover, that game loop will consume all the machine resources.

First of all we may want to control separately the period at which the game state is updated and the period at which the game is rendered to the screen. Why do we do this? Well, updating our game state at a constant rate is more important, especially if we use some physics engine. On the contrary, if our rendering is not done in time it makes no sense to render old frames while processing our game loop. We have the flexibility to skip some frames. 

## Implementation

Prior to examining the game loop, let's create the supporting classes that will form the core of the engine. We will first create an interface that will encapsulate the game logic. By doing this we will make our game engine reusable across the different chapters. This interface will have methods to initialize the game assets (`init`), handle user input (`input`), update game state (`update`) and clean up the resources (`cleanup`).

```java
package org.lwjglb.engine;

import org.lwjglb.engine.graph.Render;
import org.lwjglb.engine.scene.Scene;

public interface IAppLogic {

    void cleanup();

    void init(Window window, Scene scene, Render render);

    void input(Window window, Scene scene, long diffTimeMillis);

    void update(Window window, Scene scene, long diffTimeMillis);
}
```

As you can see, there are some classes instances which we have not defined yet (`Window`, `Scene` and `Render`) and a parameter named `diffTimeMillis` which holds the milliseconds passed between invocations of those methods.

Let's start with the `Window` class. We will encapsulate in this class all the invocations to GLFW library to create and manage a window, and its structure is like this:

```java
package org.lwjglb.engine;

import org.lwjgl.glfw.GLFWVidMode;
import org.lwjgl.system.MemoryUtil;
import org.tinylog.Logger;

import java.util.concurrent.Callable;

import static org.lwjgl.glfw.Callbacks.glfwFreeCallbacks;
import static org.lwjgl.glfw.GLFW.*;
import static org.lwjgl.opengl.GL11.*;
import static org.lwjgl.system.MemoryUtil.NULL;

public class Window {

    private final long windowHandle;
    private int height;
    private Callable<Void> resizeFunc;
    private int width;
    ...
    ...
    public static class WindowOptions {
        public boolean compatibleProfile;
        public int fps;
        public int height;
        public int ups = Engine.TARGET_UPS;
        public int width;
    }
}
```

As you can see, it defines some attributes to store the window handle, its width and height and a callback function which will be invoked nay time the window is resized. It also defines an inner class to set up some options to control window creation:

* `compatibleProfile`: This controls wether we want to use old functions from previous versions (deprecated functions) or not. 
* `fps`: Defines the target frames per second (FPS). If it has a value equal os less than zero it will mean that we do not want to set up a target but either use monitor refresh that as target FPS. In order to do so, we will use v-sync (that is the number of screen updates to wait from the time `glfwSwapBuffers` was called before swapping the buffers and returning).
* `height`: Desired window height.
* `width`: Desired window width:
* `ups`: Defines the target number of updates per second (initialized to a default value).

Let's examine the constructor of the `Window+  class:

```java
public class Window {
    ...
    public Window(String title, WindowOptions opts, Callable<Void> resizeFunc) {
        this.resizeFunc = resizeFunc;
        if (!glfwInit()) {
            throw new IllegalStateException("Unable to initialize GLFW");
        }

        glfwDefaultWindowHints();
        glfwWindowHint(GLFW_VISIBLE, GL_FALSE);
        glfwWindowHint(GLFW_RESIZABLE, GL_TRUE);

        glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
        glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 2);
        if (opts.compatibleProfile) {
            glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_COMPAT_PROFILE);
        } else {
            glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);
            glfwWindowHint(GLFW_OPENGL_FORWARD_COMPAT, GL_TRUE);
        }

        if (opts.width > 0 && opts.height > 0) {
            this.width = opts.width;
            this.height = opts.height;
        } else {
            glfwWindowHint(GLFW_MAXIMIZED, GLFW_TRUE);
            GLFWVidMode vidMode = glfwGetVideoMode(glfwGetPrimaryMonitor());
            width = vidMode.width();
            height = vidMode.height();
        }

        windowHandle = glfwCreateWindow(width, height, title, NULL, NULL);
        if (windowHandle == NULL) {
            throw new RuntimeException("Failed to create the GLFW window");
        }

        glfwSetFramebufferSizeCallback(windowHandle, (window, w, h) -> resized(w, h));

        glfwSetErrorCallback((int errorCode, long msgPtr) ->
                Logger.error("Error code [{}], msg [{]]", errorCode, MemoryUtil.memUTF8(msgPtr))
        );

        glfwSetKeyCallback(windowHandle, (window, key, scancode, action, mods) -> {
            if (key == GLFW_KEY_ESCAPE && action == GLFW_RELEASE) {
                glfwSetWindowShouldClose(window, true); // We will detect this in the rendering loop
            }
        });

        glfwMakeContextCurrent(windowHandle);

        if (opts.fps > 0) {
            glfwSwapInterval(0);
        } else {
            glfwSwapInterval(1);
        }

        glfwShowWindow(windowHandle);

        int[] arrWidth = new int[1];
        int[] arrHeight = new int[1];
        glfwGetFramebufferSize(windowHandle, arrWidth, arrHeight);
        width = arrWidth[0];
        height = arrHeight[0];
    }
    ...
}
```

We start by setting some window hints to hide the window and set it resizable. After that, we set OpenGL version and set either core or compatible profile depending on window options. Then, if we have not set a preferred width and height we get the primary monitor dimensions to set window size. We then create the window by calling the `glfwCreateWindow` and set some callbacks when window is resized or to detect window termination (when `ESC` key is pressed). If we want to manually set a target FPS, we invoke `glfwSwapInterval(0)` to disable v-sync and finally, we show the window and get the frame buffer size (the portion of the window used to render()).

The rest of the methods of the `Window` class are for cleaning up resources, the resize callback, some getters for window size and methods to poll events and to check if the window should be closed.

```java
public class Window {
    ...
    public void cleanup() {
        glfwFreeCallbacks(windowHandle);
        glfwDestroyWindow(windowHandle);
        glfwTerminate();
    }

    public int getHeight() {
        return height;
    }

    public int getWidth() {
        return width;
    }

    public boolean isKeyPressed(int keyCode) {
        return glfwGetKey(windowHandle, keyCode) == GLFW_PRESS;
    }

    public void pollEvents() {
        glfwPollEvents();
    }

    protected void resized(int width, int height) {
        this.width = width;
        this.height = height;
        try {
            resizeFunc.call();
        } catch (Exception excp) {
            Logger.error("Error calling resize callback", excp);
        }
    }

    public void update() {
        glfwSwapBuffers(windowHandle);
    }

    public boolean windowShouldClose() {
        return glfwWindowShouldClose(windowHandle);
    }
    ...
}
```

The `Scene` class will hold 3D scene future elements (models, etc.). By now it is just an empty place holder:

```java
package org.lwjglb.engine.scene;

public class Scene {

    public Scene() {
    }

    public void cleanup() {
        // Nothing to be done here yet
    }
}
```

The `Render` class is just now another place holder that just clears the screen:

```java
package org.lwjglb.engine.graph;

import org.lwjgl.opengl.GL;
import org.lwjglb.engine.Window;
import org.lwjglb.engine.scene.Scene;

import static org.lwjgl.opengl.GL11.*;

public class Render {

    public Render() {
        GL.createCapabilities();
    }

    public void cleanup() {
        // Nothing to be done here yet
    }

    public void render(Window window, Scene scene) {
        glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
    }
}
```

Now we can implement the game loop in a new class named `Engine` which starts like this:

```java
package org.lwjglb.engine;

import org.lwjglb.engine.graph.Render;
import org.lwjglb.engine.scene.Scene;

public class Engine {

    public static final int TARGET_UPS = 30;
    private final IAppLogic appLogic;
    private final Window window;
    private Render render;
    private boolean running;
    private Scene scene;
    private int targetFps;
    private int targetUps;

    public Engine(String windowTitle, Window.WindowOptions opts, IAppLogic appLogic) {
        window = new Window(windowTitle, opts, () -> {
            resize();
            return null;
        });
        targetFps = opts.fps;
        targetUps = opts.ups;
        this.appLogic = appLogic;
        render = new Render();
        scene = new Scene();
        appLogic.init(window, scene, render);
        running = true;
    }

    private void cleanup() {
        appLogic.cleanup();
        render.cleanup();
        scene.cleanup();
        window.cleanup();
    }

    private void resize() {
        // Nothing to be done yet
    }
    ...
}
```

The `Engine`class, receives in the constructor the title of the window, the window options and a reference to the implementation of the `IAppLogic` interface.  In the constructor it creates instance of the `Window`, `Render` and `Scene` classes. The `cleanup` method just invokes the other classes `cleanup` resources. The game loop is defined in the `run` method which is defined like this:

```java
public class Engine {
    ...
    private void run() {
        long initialTime = System.currentTimeMillis();
        float timeU = 1000.0f / targetUps;
        float timeR = targetFps > 0 ? 1000.0f / targetFps : 0;
        float deltaUpdate = 0;
        float deltaFps = 0;

        long updateTime = initialTime;
        while (running && !window.windowShouldClose()) {
            window.pollEvents();

            long now = System.currentTimeMillis();
            deltaUpdate += (now - initialTime) / timeU;
            deltaFps += (now - initialTime) / timeR;

            if (targetFps <= 0 || deltaFps >= 1) {
                appLogic.input(window, scene, now - initialTime);
            }

            if (deltaUpdate >= 1) {
                long diffTimeMillis = now - updateTime;
                appLogic.update(window, scene, diffTimeMillis);
                updateTime = now;
                deltaUpdate--;
            }

            if (targetFps <= 0 || deltaFps >= 1) {
                render.render(window, scene);
                deltaFps--;
                window.update();
            }
            initialTime = now;
        }

        cleanup();
    }
    ...
}
```

The loop starts by calculating two parameters: `timeU` and `timeR` which control the maximum elapsed time between updates (`timeU`) and render calls (`timeR`) in milliseconds. If those periods are consumed we need either to update game state or to render. In the later case, if the target FPS is set to 0 we will rely on  v-sync refresh rate so we just set tha value to `0`. The loop starts by polling the events over the window, after that, we get current time in milliseconds. After that we get the elapsed time between update and render calls. If we have passed the maximum elapsed time for render (or relay in v-sync), we process user input by calling `appLogic.input`. If we have surpassed maximum update elapsed time we update game state by calling `appLogic.update`.  we have passed the maximum elapsed time for render (or relay in v-sync), we trigger render calls by calling `render.render`.

At the end of the loop we call the `cleanup` method to free resources.

Finally the `Engine` is completed like this:

```java
public class Engine {
    ...
    public void start() {
        running = true;
        run();
    }

    public void stop() {
        running = false;
    }
}
```

A little bit note on threading. GLFW requires to be initialized from the main thread. Polling of events should also be done in that thread. Therefore, instead of creating a separate thread for the game loop, which is what you would see commonly in games, we will execute everything from the main thread. This is whey we do not create new `Thread` in the `start` method.

Finally, we just simplify the `Main` class to this:

```java
package org.lwjglb.game;

import org.lwjglb.engine.*;
import org.lwjglb.engine.graph.Render;
import org.lwjglb.engine.scene.Scene;

public class Main implements IAppLogic {

    public static void main(String[] args) {
        Main main = new Main();
        Engine gameEng = new Engine("chapter-02", new Window.WindowOptions(), main);
        gameEng.start();
    }

    @Override
    public void cleanup() {
        // Nothing to be done yet
    }

    @Override
    public void init(Window window, Scene scene, Render render) {
        // Nothing to be done yet
    }

    @Override
    public void input(Window window, Scene scene, long diffTimeMillis) {
        // Nothing to be done yet
    }

    @Override
    public void update(Window window, Scene scene, long diffTimeMillis) {
        // Nothing to be done yet
    }
}
```

We just create the `Engine` instance and start it up in the `main` method. The `Main` class also implements the `IAppLogic` interface which by now is just empty.

[Next chapter](../chapter-03/chapter-03.md)