# Chapter 09 - Loading more complex models: Assimp

The capability of loading complex 3D models in different formats is crucial in order to write a game. The task of writing parsers for some of them would require lots of work. Even just supporting a single format can be time consuming. Fortunately, the [Assimp](http://assimp.sourceforge.net/) library already can be used to parse many common 3D formats. It’s a C/C++ library which can load static and animated models in a variety of formats. LWJGL provides the bindings to use them from Java code. In this chapter, we will explain how it can be used.


You can find the complete source code for this chapter [here](https://github.com/lwjglgamedev/lwjglbook/tree/main/chapter-09).

## Model loader

The first thing is adding Assimp maven dependencies to the project pom.xml. We need to add compile time and runtime dependencies.

```xml
<dependency>
    <groupId>org.lwjgl</groupId>
    <artifactId>lwjgl-assimp</artifactId>
    <version>${lwjgl.version}</version>
</dependency>
<dependency>
    <groupId>org.lwjgl</groupId>
    <artifactId>lwjgl-assimp</artifactId>
    <version>${lwjgl.version}</version>
    <classifier>${native.target}</classifier>
    <scope>runtime</scope>
</dependency>
```

Once the dependencies has been set, we will create a new class named `ModelLoader` that will be used to load models with Assimp. The class defines two static public methods:

```java
package org.lwjglb.engine.scene;

import org.joml.Vector4f;
import org.lwjgl.PointerBuffer;
import org.lwjgl.assimp.*;
import org.lwjgl.system.MemoryStack;
import org.lwjglb.engine.graph.*;

import java.io.File;
import java.nio.IntBuffer;
import java.util.*;

import static org.lwjgl.assimp.Assimp.*;

public class ModelLoader {

    private ModelLoader() {
        // Utility class
    }

    public static Model loadModel(String modelId, String modelPath, TextureCache textureCache) {
        return loadModel(modelId, modelPath, textureCache, aiProcess_GenSmoothNormals | aiProcess_JoinIdenticalVertices |
                aiProcess_Triangulate | aiProcess_FixInfacingNormals | aiProcess_CalcTangentSpace | aiProcess_LimitBoneWeights |
                aiProcess_PreTransformVertices);

    }

    public static Model loadModel(String modelId, String modelPath, TextureCache textureCache, int flags) {
        ...

    }
    ...
}
```

Both methods have the following arguments:

* `modelId`: A unique identifier for the model to be loaded.

* `modelPath`: The path to the file where the model file is located. This is a regular file path, no CLASSPATH relative paths, because Assimp may need to load additional files and may use the same base path as the `modelPath` \(For instance, material files for wavefront, OBJ, files\). If you embed your resources inside a JAR file, Assimp will not be able to import it, so it must be a file system path. When loading textures we will use `modelPath` to get the base directory where the model is located to load textures (overriding whatever path is defined in the model). We do this because some models contain absolute paths to local folders of where the model was developed which, obviously, are not accessible.

* `textureCache`: A reference to the texture cache to avoid loading the same texture multiple times.

The second method has an extra argument named `flags`. This parameter allows to tune the loading process. The first method invokes the second one and passes some values that are useful in most of the situations:

* `aiProcess_JoinIdenticalVertices`: This flag reduces the number of vertices that are used, identifying those that can be reused between faces.

* `aiProcess_Triangulate`: The model may use quads or other geometries to define their elements. Since we are only dealing with triangles, we must use this flag to split all he faces into triangles \(if needed\).

* `aiProcess_FixInfacingNormals`: This flags try to reverse normals that may point inwards.

* `aiProcess_CalcTangentSpace`: We will use this parameter when implementing lights, but it basically calculates tangent and bitangents using normals information.  

* `aiProcess_LimitBoneWeights`: We will use this parameter when implementing animations, but it basically limit the number of weights that affect a single vertex.

* `aiProcess_PreTransformVertices`: This flag performs some transformation over the data loaded so the model is placed in the origin and the coordinates are corrected to math OpenGL coordinate System. If you have problems with models that are rotated, make sure to use this flag. Important: do not use this flag if your model uses animations, this flag will remove that information.

There are many other flags that can be used, you can check them in the LWJGL or Assimp documentation.

Let’s go back to the second constructor. The first thing we do is invoke the `aiImportFile` method to load the model with the selected flags.

```java
public class ModelLoader {
    ...
    public static Model loadModel(String modelId, String modelPath, TextureCache textureCache, int flags) {
        File file = new File(modelPath);
        if (!file.exists()) {
            throw new RuntimeException("Model path does not exist [" + modelPath + "]");
        }
        String modelDir = file.getParent();

        AIScene aiScene = aiImportFile(modelPath, flags);
        if (aiScene == null) {
            throw new RuntimeException("Error loading model [modelPath: " + modelPath + "]");
        }
        ...
    }
    ...
}
```

The rest of the code for the constructor is as follows:

```java
public class ModelLoader {
    ...
    public static Model loadModel(String modelId, String modelPath, TextureCache textureCache, int flags) {
        ...
        int numMaterials = aiScene.mNumMaterials();
        List<Material> materialList = new ArrayList<>();
        for (int i = 0; i < numMaterials; i++) {
            AIMaterial aiMaterial = AIMaterial.create(aiScene.mMaterials().get(i));
            materialList.add(processMaterial(aiMaterial, modelDir, textureCache));
        }

        int numMeshes = aiScene.mNumMeshes();
        PointerBuffer aiMeshes = aiScene.mMeshes();
        Material defaultMaterial = new Material();
        for (int i = 0; i < numMeshes; i++) {
            AIMesh aiMesh = AIMesh.create(aiMeshes.get(i));
            Mesh mesh = processMesh(aiMesh);
            int materialIdx = aiMesh.mMaterialIndex();
            Material material;
            if (materialIdx >= 0 && materialIdx < materialList.size()) {
                material = materialList.get(materialIdx);
            } else {
                material = defaultMaterial;
            }
            material.getMeshList().add(mesh);
        }

        if (!defaultMaterial.getMeshList().isEmpty()) {
            materialList.add(defaultMaterial);
        }

        return new Model(modelId, materialList);
    }
    ...
}
```

We process the materials contained in the model. Materials define color and textures to be used by the meshes that compose the model. Then we process the different meshes. A model can define several meshes and each of them can use one of the materials defined for the model. This is why we process meshes after materials and link to them, to avoid repeating binding calls when rendering.

If you examine the code above you may see that many of the calls to the Assimp library return `PointerBuffer` instances. You can think about them like C pointers, they just point to a memory region which contain data. You need to know in advance the type of data that they hold in order to process them. In the case of materials, we iterate over that buffer creating instances of the `AIMaterial` class. In the second case, we iterate over the buffer that holds mesh data creating instance of the `AIMesh` class.

Let’s examine the `processMaterial` method.

```java
public class ModelLoader {
    ...
    private static Material processMaterial(AIMaterial aiMaterial, String modelDir, TextureCache textureCache) {
        Material material = new Material();
        try (MemoryStack stack = MemoryStack.stackPush()) {
            AIColor4D color = AIColor4D.create();

            int result = aiGetMaterialColor(aiMaterial, AI_MATKEY_COLOR_DIFFUSE, aiTextureType_NONE, 0,
                    color);
            if (result == aiReturn_SUCCESS) {
                material.setDiffuseColor(new Vector4f(color.r(), color.g(), color.b(), color.a()));
            }

            AIString aiTexturePath = AIString.calloc(stack);
            aiGetMaterialTexture(aiMaterial, aiTextureType_DIFFUSE, 0, aiTexturePath, (IntBuffer) null,
                    null, null, null, null, null);
            String texturePath = aiTexturePath.dataString();
            if (texturePath != null && texturePath.length() > 0) {
                material.setTexturePath(modelDir + File.separator + new File(texturePath).getName());
                textureCache.createTexture(material.getTexturePath());
                material.setDiffuseColor(Material.DEFAULT_COLOR);
            }

            return material;
        }
    }
    ...
}
```

We first get the material color, in this case the diffuse color (by setting the `AI_MATKEY_COLOR_DIFFUSE` flag). There are many different types of colors which we will use when applying lights, for example we have  diffuse, ambient (for ambient light), specular (for specular factor of lights, etc.) After that, we check if the material defines a texture or not. If so, that is if there is a texture path, we store the texture path and delegate texture creation to the `TexturCache` class as in previous examples. In this case, if the material defines a texture we set the diffuse color to a default value, which is black. By doing this we will be able to use both values, diffuse color and texture without checking if there is a texture or not. If the model does not define a texture we will use a default black texture which can be combined with the material color.

The `processMesh` method is defined like this.

```java
public class ModelLoader {
    ...
    private static Mesh processMesh(AIMesh aiMesh) {
        float[] vertices = processVertices(aiMesh);
        float[] textCoords = processTextCoords(aiMesh);
        int[] indices = processIndices(aiMesh);

        // Texture coordinates may not have been populated. We need at least the empty slots
        if (textCoords.length == 0) {
            int numElements = (vertices.length / 3) * 2;
            textCoords = new float[numElements];
        }

        return new Mesh(vertices, textCoords, indices);
    }
    ...
}
```

A `Mesh` is defined by a set of vertices position,  texture coordinates and indices. Each of these elements are processed in the `processVertices`,  `processTextCoords` and `processIndices` methods. After processing all that data we check if texture coordinates have been defined. If not, we just assign a set of texture coordinates to 0.0f to ensure consistency of the VAO.

The `processXXX` methods are very simple, they just invoke the corresponding method over the `AIMesh` instance that returns the desired data and store it into an array:

```java
public class ModelLoader {
    ...
    private static int[] processIndices(AIMesh aiMesh) {
        List<Integer> indices = new ArrayList<>();
        int numFaces = aiMesh.mNumFaces();
        AIFace.Buffer aiFaces = aiMesh.mFaces();
        for (int i = 0; i < numFaces; i++) {
            AIFace aiFace = aiFaces.get(i);
            IntBuffer buffer = aiFace.mIndices();
            while (buffer.remaining() > 0) {
                indices.add(buffer.get());
            }
        }
        return indices.stream().mapToInt(Integer::intValue).toArray();
    }
    ...
    private static float[] processTextCoords(AIMesh aiMesh) {
        AIVector3D.Buffer buffer = aiMesh.mTextureCoords(0);
        if (buffer == null) {
            return new float[]{};
        }
        float[] data = new float[buffer.remaining() * 2];
        int pos = 0;
        while (buffer.remaining() > 0) {
            AIVector3D textCoord = buffer.get();
            data[pos++] = textCoord.x();
            data[pos++] = 1 - textCoord.y();
        }
        return data;
    }

    private static float[] processVertices(AIMesh aiMesh) {
        AIVector3D.Buffer buffer = aiMesh.mVertices();
        float[] data = new float[buffer.remaining() * 3];
        int pos = 0;
        while (buffer.remaining() > 0) {
            AIVector3D textCoord = buffer.get();
            data[pos++] = textCoord.x();
            data[pos++] = textCoord.y();
            data[pos++] = textCoord.z();
        }
        return data;
    }
}
```

You can see that get get a buffer to the vertices by invoking the `mVertices` method. We just simply process them to create a `List` of floats that contain the vertices positions. Since, the method returns just a buffer you could pass that information directly to the OpenGL methods that create vertices. We do not do it that way for two reasons. The first one is try to reduce as much as possible the modifications over the code base. Second one is that by loading into an intermediate structure you may be able to perform some pros-processing tasks and even debug the loading process.

If you want a sample of the much more efficient approach, that is, directly passing the buffers to OpenGL, you can check this [sample](https://github.com/LWJGL/lwjgl3-demos/blob/master/src/org/lwjgl/demo/opengl/assimp/WavefrontObjDemo.java).

## Using the models

We need to modify the `Material` class to add support for diffuse color:

```java
public class Material {
 
    public static final Vector4f DEFAULT_COLOR = new Vector4f(0.0f, 0.0f, 0.0f, 1.0f);

    private Vector4f diffuseColor;
    ...
    public Material() {
        diffuseColor = DEFAULT_COLOR;
        ...
    }
    ...
    public Vector4f getDiffuseColor() {
        return diffuseColor;
    }
    ...
    public void setDiffuseColor(Vector4f diffuseColor) {
        this.diffuseColor = diffuseColor;
    }
    ...
}
```

In the `SceneRender` class, we need to create, and properly set up while rendering, the material diffuse color:
```java
public class SceneRender {
    ...
    private void createUniforms() {
        ...
        uniformsMap.createUniform("material.diffuse");
    }

    public void render(Scene scene) {
        ...
        for (Model model : models) {
            List<Entity> entities = model.getEntitiesList();

            for (Material material : model.getMaterialList()) {
                uniformsMap.setUniform("material.diffuse", material.getDiffuseColor());
                ...
            }
        }
        ...
    }
    ...
}
```

As you can see we are using a weird name for the uniform with a `.` in the name. This is because we will use structures in the shader. With structures we can group several types into a single combined one. You can see this in the fragment shader:

```glsl
#version 330

in vec2 outTextCoord;

out vec4 fragColor;

struct Material
{
    vec4 diffuse;
};

uniform sampler2D txtSampler;
uniform Material material;

void main()
{
    fragColor = texture(txtSampler, outTextCoord) + material.diffuse;
}
```

We will need also to add a new method to the `UniformsMap` class to add support for passing `Vector4f` values
```java
public class UniformsMap {
    ...
    public void setUniform(String uniformName, Vector4f value) {
        glUniform4f(getUniformLocation(uniformName), value.x, value.y, value.z, value.w);
    }
}
```

Finally, we need to modify the `Main` class to use the `ModelLoader` class to load models:
```java
public class Main implements IAppLogic {
    ...
    public static void main(String[] args) {
        ...
        Engine gameEng = new Engine("chapter-09", new Window.WindowOptions(), main);
        ...
    }
    ...
    public void init(Window window, Scene scene, Render render) {
        Model cubeModel = ModelLoader.loadModel("cube-model", "resources/models/cube/cube.obj",
                scene.getTextureCache());
        scene.addModel(cubeModel);

        cubeEntity = new Entity("cube-entity", cubeModel.getId());
        cubeEntity.setPosition(0, 0, -2);
        scene.addEntity(cubeEntity);
    }
    ...
}
```

As you can see, the `init` method has been simplified a lot, no more model data embedded in the code. Now we are using a cube model which uses the wavefront format. You can locate model files in the `resources\models\cube` folder. You will find there, the following files:

* `cube.obj`: The main model file. In fact is a text based format, so you can open it and see how vertices, indices and textures coordinates are defined and glued together by defining faces. It also contains a reference to a material file.

* `cube.mtl`: The material file, it defines colors and textures.

* `cube.png`: The texture file of the model.

Finally, we will add another feature to optimize the render. We will reduce the amount of data that is being rendered by applying face culling. As you well know, a cube is made of six faces and we are rendering the six faces they are not visible. You can check this if you zoom inside a cube, you will see its interior.

Faces that cannot be seen should be discarded immediately and this is what face culling does. In fact, for a cube you can only see 3 faces at the same time, so we can just discard half of the faces just by applying face culling \(this will only be valid if your game does not require you to dive into the inner side of a model\).

For every triangle, face culling checks if it's facing towards us and discards the ones that are not facing that direction. But, how do we know if a triangle is facing towards us or not? Well, the way that OpenGL does this is by the winding order of the vertices that compose a triangle.

Remember from the first chapters that we may define the vertices of a triangle in clockwise or counter-clockwise order. In OpenGL, by default, triangles that are in counter-clockwise order are facing towards the viewer and triangles that are in clockwise order are facing backwards. The key thing here, is that this order is checked while rendering taking into consideration the point of view. So a triangle that has been defined in counter-clock wise order can be interpreted, at rendering time, as being defined clockwise because of the point of view.

We will enable face culling in the `Render` class:

```java
public class Render {
    ...
    public Render() {
        ...
        glEnable(GL_CULL_FACE);
        glCullFace(GL_BACK);
        ...
    }
    ...
}
```

The first line will enable face culling and the second line states that faces that are facing backwards should be culled \(removed\).


If you run the sample you will see the same result as in previous chapter, however, if you zoom in into the cube, inner faces will not be rendered. You can modify this sample to load more complex models.
 
[Next chapter](../chapter-10/chapter-10.md)
