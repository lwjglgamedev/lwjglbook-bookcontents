# Chapter 08 - Camera

In this chapter we will learn how to move inside a rendered 3D scene. This capability is like having a camera that can travel inside the 3D world and in fact that's the term used to refer to it.

You can find the complete source code for this chapter [here](https://github.com/lwjglgamedev/lwjglbook/chapter-08).

## Camera introduction

If you try to search for specific camera functions in OpenGL you will discover that there is no camera concept, or in other words the camera is always fixed, centered in the \(0, 0, 0\) position at the center of the screen. So what we will do is a simulation that gives us the impression that we have a camera capable of moving inside the 3D scene. How do we achieve this? Well, if we cannot move the camera then we must move all the objects contained in our 3D space at once. In other words, if we cannot move a camera we will move the whole world.

Hence, suppose that we would like to move the camera position along the z axis from a starting position \(Cx, Cy, Cz\) to a position \(Cx, Cy, Cz+dz\) to get closer to the object which is placed at the coordinates \(Ox, Oy, Oz\).

![Camera movement](camera_movement.png)

What we will actually do is move the object \(all the objects in our 3D space indeed\) in the opposite direction that the camera should move. Think about it like the objects being placed in a treadmill.

![Actual movement](actual_movement.png)

A camera can be displaced along the three axis \(x, y and z\) and also can rotate along them \(roll, pitch and yaw\).

![Roll pitch and yaw](roll_pitch_yaw.png)

So basically what we must do is to be able to move and rotate all of the objects of our 3D world. How are we going to do this? The answer is to apply another transformation that will translate all of the vertices of all of the objects in the opposite direction of the movement of the camera and that will rotate them according to the camera rotation. This will be done of course with another matrix, the so called view matrix. This matrix will first perform the translation and then the rotation along the axis.

Let's see how we can construct that matrix. If you remember from the transformations chapter our transformation equation was like this:

$$
\begin{array}{lcl}
Transf & = & \lbrack ProjMatrix \rbrack \cdot \lbrack TranslationMatrix \rbrack \cdot \lbrack  RotationMatrix \rbrack \cdot \lbrack  ScaleMatrix \rbrack \\ 
 & = & \lbrack   ProjMatrix \rbrack  \cdot \lbrack  WorldMatrix \rbrack
\end{array}
$$

The view matrix should be applied before multiplying by the projection matrix, so our equation should be now like this:

$$
\begin{array}{lcl}
Transf & = & \lbrack  ProjMatrix \rbrack \cdot \lbrack  ViewMatrix \rbrack \cdot \lbrack  TranslationMatrix \rbrack \cdot \lbrack  RotationMatrix \rbrack \cdot \lbrack ScaleMatrix \rbrack \\
  & = & \lbrack ProjMatrix \rbrack \cdot \lbrack  ViewMatrix \rbrack \cdot \lbrack  WorldMatrix \rbrack 
\end{array}
$$

## Camera implementation

So let’s start modifying our code to support a camera. First of all we will create a new class called `Camera` which will hold the position and rotation state of our camera as wll as its view matrix. The class is defined like this:
```java
package org.lwjglb.engine.scene;

import org.joml.*;

public class Camera {

    private Vector3f direction;
    private Vector3f position;
    private Vector3f right;
    private Vector2f rotation;
    private Vector3f up;
    private Matrix4f viewMatrix;

    public Camera() {
        direction = new Vector3f();
        right = new Vector3f();
        up = new Vector3f();
        position = new Vector3f();
        viewMatrix = new Matrix4f();
        rotation = new Vector2f();
    }

    public void addRotation(float x, float y) {
        rotation.add(x, y);
        recalculate();
    }

    public Vector3f getPosition() {
        return position;
    }

    public Matrix4f getViewMatrix() {
        return viewMatrix;
    }

    public void moveBackwards(float inc) {
        viewMatrix.positiveZ(direction).negate().mul(inc);
        position.sub(direction);
        recalculate();
    }

    public void moveDown(float inc) {
        viewMatrix.positiveY(up).mul(inc);
        position.sub(up);
        recalculate();
    }

    public void moveForward(float inc) {
        viewMatrix.positiveZ(direction).negate().mul(inc);
        position.add(direction);
        recalculate();
    }

    public void moveLeft(float inc) {
        viewMatrix.positiveX(right).mul(inc);
        position.sub(right);
        recalculate();
    }

    public void moveRight(float inc) {
        viewMatrix.positiveX(right).mul(inc);
        position.add(right);
        recalculate();
    }

    public void moveUp(float inc) {
        viewMatrix.positiveY(up).mul(inc);
        position.add(up);
        recalculate();
    }

    private void recalculate() {
        viewMatrix.identity()
                .rotateX(rotation.x)
                .rotateY(rotation.y)
                .translate(-position.x, -position.y, -position.z);
    }

    public void setPosition(float x, float y, float z) {
        position.set(x, y, z);
        recalculate();
    }

    public void setRotation(float x, float y) {
        rotation.set(x, y);
        recalculate();
    }
}
```

As you can see, besides rotation and position we define some vectors to define forward up and right directions. This is because we are implementing a free space movement camera, and whe whe rotate it if we want to move forward we just want to move where the camera is pointing, not to predefined axis. We need to get those vectors to calculate where the next position will be placed. And finally, at the end the state of the camera is stored into a 4x4 matrix, the view matrix, so any time we change position or rotation we need to update it. As you can see, when updating thew view matrix, we first need to do the rotation and then the translation. If we did the opposite we would not be rotating along the camera position but along the coordinates origin. 

The `Camera` class also provides methods tup update position when moving forward, up or to the right. In these methods, the view matrix is used to calculate where the forward, up or right methods should be according to current state, and increases the position accordingly. We use the fantastic JOML library to these calculations for us while maintaining the code quite simple.

## Using the Camera

We will store a `Camera` instance in the `Scene` class, so let's go for the changes:

```java
public class Scene {
    ...
    private Camera camera;
    ...
    public Scene(int width, int height) {
        ...
        camera = new Camera();
    }
    ...
    public Camera getCamera() {
        return camera;
    }
    ...
}
```

It would be nice to control de camera with our mouse. In order to do so, we will create a new class to handle mouse events so we can use them to update camera rotation. Here's the code for that class.

```java
package org.lwjglb.engine;

import org.joml.Vector2f;

import static org.lwjgl.glfw.GLFW.*;

public class MouseInput {

    private Vector2f currentPos;
    private Vector2f displVec;
    private boolean inWindow;
    private boolean leftButtonPressed;
    private Vector2f previousPos;
    private boolean rightButtonPressed;

    public MouseInput(long windowHandle) {
        previousPos = new Vector2f(-1, -1);
        currentPos = new Vector2f();
        displVec = new Vector2f();
        leftButtonPressed = false;
        rightButtonPressed = false;
        inWindow = false;

        glfwSetCursorPosCallback(windowHandle, (handle, xpos, ypos) -> {
            currentPos.x = (float) xpos;
            currentPos.y = (float) ypos;
        });
        glfwSetCursorEnterCallback(windowHandle, (handle, entered) -> inWindow = entered);
        glfwSetMouseButtonCallback(windowHandle, (handle, button, action, mode) -> {
            leftButtonPressed = button == GLFW_MOUSE_BUTTON_1 && action == GLFW_PRESS;
            rightButtonPressed = button == GLFW_MOUSE_BUTTON_2 && action == GLFW_PRESS;
        });
    }

    public Vector2f getCurrentPos() {
        return currentPos;
    }

    public Vector2f getDisplVec() {
        return displVec;
    }

    public void input() {
        displVec.x = 0;
        displVec.y = 0;
        if (previousPos.x > 0 && previousPos.y > 0 && inWindow) {
            double deltax = currentPos.x - previousPos.x;
            double deltay = currentPos.y - previousPos.y;
            boolean rotateX = deltax != 0;
            boolean rotateY = deltay != 0;
            if (rotateX) {
                displVec.y = (float) deltax;
            }
            if (rotateY) {
                displVec.x = (float) deltay;
            }
        }
        previousPos.x = currentPos.x;
        previousPos.y = currentPos.y;
    }

    public boolean isLeftButtonPressed() {
        return leftButtonPressed;
    }

    public boolean isRightButtonPressed() {
        return rightButtonPressed;
    }
}
```

The `MouseInput` class, in its constructor, registers a set of callbacks to process mouse events:

* `glfwSetCursorPosCallback`: Registers a callback that will be invoked when the mouse is moved.
* `glfwSetCursorEnterCallback`: Registers a callback that will be invoked when the mouse enters our window. We will be receiving mouse events even if the mouse is not in our window. We use this callback to track when the mouse is in our window.
* `glfwSetMouseButtonCallback`: Registers a callback that will be invoked when a mouse button is pressed.

The `MouseInput` class provides an input method which should be called when game input is processed. This method calculates the mouse displacement from the previous position and stores it into the `displVec` variable so it can be used by our game.

The `MouseInput` class will be instantiated in our `Window` class, which will also provide a getter to return its instance. It also invokes the input method each time events are polled.

```java
public class Window {
    ...
    private MouseInput mouseInput;
    ...
    public Window(String title, WindowOptions opts, Callable<Void> resizeFunc) {
        ...
        mouseInput = new MouseInput(windowHandle);
    }
    ...
    public MouseInput getMouseInput() {
        return mouseInput;
    }
    ...    
    public void pollEvents() {
        ...
        mouseInput.input();
    }
    ...
}
```

Now we can modify the vertex shader to use the `Camera`'s view matrix, which as you may guess will be passed as an uniform.

```glsl
#version 330

layout (location=0) in vec3 position;
layout (location=1) in vec2 texCoord;

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

So the next step is to properly create the uniform in the `SceneRender` class and update its value in each `render` call:

```java
public class SceneRender {
    ...
    private void createUniforms() {
        ...
        uniformsMap.createUniform("viewMatrix");
        ...
    }    
    ...
    public void render(Scene scene) {
        ...
        uniformsMap.setUniform("projectionMatrix", scene.getProjection().getProjMatrix());
        uniformsMap.setUniform("viewMatrix", scene.getCamera().getViewMatrix());
        ...
    }
}
```

And that’s all, our base code supports the concept of a camera. Now we need to use it. We can change the way we handle the input and update the camera. We will set the following controls:

* Keys “A” and “D” to move the camera to the left and right \(x axis\) respectively.
* Keys “W” and “S” to move the camera forward and backwards \(z axis\) respectively.
* Keys “Z” and “X” to move the camera up and down \(y axis\) respectively.

We will use the mouse position to rotate the camera along the x and y axis when the right button of the mouse is pressed.  

Now we are ready to update our `Main` class to process the keyboard and mouse input.

```java

public class Main implements IAppLogic {

    private static final float MOUSE_SENSITIVITY = 0.1f;
    private static final float MOVEMENT_SPEED = 0.005f;
    ...

    public static void main(String[] args) {
        ...
        Engine gameEng = new Engine("chapter-08", new Window.WindowOptions(), main);
        ...
    }
    ...
    public void input(Window window, Scene scene, long diffTimeMillis) {
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
        if (window.isKeyPressed(GLFW_KEY_UP)) {
            camera.moveUp(move);
        } else if (window.isKeyPressed(GLFW_KEY_DOWN)) {
            camera.moveDown(move);
        }

        MouseInput mouseInput = window.getMouseInput();
        if (mouseInput.isRightButtonPressed()) {
            Vector2f displVec = mouseInput.getDisplVec();
            camera.addRotation((float) Math.toRadians(-displVec.x * MOUSE_SENSITIVITY),
                    (float) Math.toRadians(-displVec.y * MOUSE_SENSITIVITY));
        }
    }
    ...
}
```

[Next chapter](../chapter-09/chapter-09.md)