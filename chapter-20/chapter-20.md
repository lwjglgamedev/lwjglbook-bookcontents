# Chapter 20 - Indirect drawing (static models)

Until this chapter, we have rendered the models by binding their material uniforms, their textures, their vertices and indices buffers and submitting one draw command for each of the meshes they are composed. In this chapter, we will start our way to a more efficient wat of rendering, we will begin the implementation of a bind-less render (at aleast almost bind-less). In this type of rendering we do not invoke a bunch of draw commands to draw the scene, instead we populate a buffer with the instructions that will allow the GPU to render them. This is called indirect rendering and it is a more efficient way of drawing because:

- We remove the need to perform several bind operations before drawing each mesh.
- We need just to invoke a single draw call.
- We can perform in-GPU operations, such as frustum culling reducing the load on the CPU side.

As you can see, the ultimate goal is to maximize the utilization of the GPU while removing potential bottlenecks that may occur at the CPU side and latencies due to CPU to GPU communications. In this chapter we will transform our render to use indirect drawing starting with just static models. Animated models will be handled in next chapter.

You can find the complete source code for this chapter [here](https://github.com/lwjglgamedev/lwjglbook/tree/main/chapter-20).

## Concepts

Prior to explaining the code, let's explain the concepts behind indirect drawing. In essence, we need to create a buffer which stores the drawing parameters which wll be used to render the vertices. You can think about that as instruction blocks, or draw commands, that will be read by the GPU that will instruct it to perform the drawing. Once the buffer is populated, we invoke the `glMultiDrawElementsIndirect` to trigger that process. Each draw command stored in the buffer is defined by the following parameters (if you are using C, this is modelled by the `DrawElementsIndirectCommand` structure):

* `count`: The number of vertices to be drawn (understanding a vertex as the structure which groups the position, normal information, texture coordinates, etc.). This should contain the same values as the number of vertices which we used when invoking the `glDrawElements` when rendering meshes.
* `instanceCount`: The number of instances to be drawn. We may have several entities that share the same model. Instead of storing a drawing instruction for each entity, we can just submit a single draw instruction but setting the number of entities that we want to draw. This is called instance rendering, and will save a lot of computing time. Without indirect drawing you can achieve the same results by setting specific attributes per VAO. I think that it is even simpler with this technique.
* `firstIndex`: An offset to the buffer that will hold the indices values used for this draw instructions (the offset is measured in number of indices, not a byte offset). 
* `baseVertex`: An offset to the buffer that will hold the vertices data (the offset is measured on number of vertices, not a byte offset).
* `baseInstance`: We can use this parameter to set a value that will be shared by all the instances to be drawn. Combining this value with the number of the instance to be drawn we will be able to access per instance data (we will see this later on).

Although it has been already commented when describing the parameters, indirect drawing needs a buffer that will hold the vertices data and another one for the indices. Th difference is that we will need to combine all that data form the multiple meshes that conform the models of our scene into a single buffer. The way we will access per-mesh specific data is by playing tih the offset values of the drawing parameters.

Another aspect to solve is how we pass material information or per-entity data (such as model matrices). In previous chapters we used uniforms for that, setting the proper value when we changed the mesh or the entities to be drawn. With indirect drawing we cannot do that, we cannot modify data during the render process, since submit a bulk set of drawing instructions at once. The solution to that is to use additional buffers, we can store per-entity data in a buffer and use the `baseInstance` parameter (combined with the instance id) to access the proper data (per entity) inside that buffer (we will see later on, that instead of a buffer we will use an array of uniforms, but you could use also a simpler buffer for that). Inside that buffer we will hold indices to access two additional buffers:

* One that will hold model matrices data.
* One that will hold material data (albedo color, etc.).

For textures we will use an array of textures which should not be confused with an array texture. An array texture is a texture which contains an array of values with texture information, with multiple images of the same size. An array of texture is a list of samples which map to regular textures, therefore they can have different sizes. Arrays of textures have a limitation, its length cannot have arbitrary length, they have a limit, that in teh examples we will set up to 16 textures (although you may want to check the capabilities of your GPU prior to setting that limit). 16 texture is not a hig value if you are using multiple models, in order to circumvent this limitation you may have two options:

* Use a texture atlas (a giant texture file which combines individual textures). Even if you are not using indirect drawing you should try to use texture atlas as much as possible, since it limits the binding calls.
* Use bindless textures. This approach basically allows us to pass handles (64 bit integer values) to identify a texture and use that identifier to get a sampler withing the shader program. This should be definitely the way to go with indirect rendering if you can (this is not a core feature but an extension starting with 4.4 version). We will no use this approach because RenderDoc does not currently support this (loosing the capability of debugging without RenderDoc is a showstopper for me).

The following picture depicts the buffers and structures involved in indirect drawing (keep in mind that this is only valid while rendering static models. We will see the new structures that we need to use when rendering animated models in the next chapter).

![Indirect drawing](indirect-drawing.svg)

Please keep in mind that we will use arrays of uniforms for per entity-data, materials and model matrices (at the end an array is a buffer, but we will be able to access the dat in handy way by using uniforms).

## Implementation

In order to use indirect drawing we will need to use at least OpenGL version 4.6. Therefore, the first step is to update the major and minor versions we use as window hints for window creation:

```java
public class Window {
    ...
    public Window(String title, WindowOptions opts, Callable<Void> resizeFunc) {
        ...
        glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 4);
        glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 6);
        ...
    }
    ...
}
```

The next step is to modify the code to load all the meshes into a single buffer, but, prior to that, wew ill modify the class hierarchy that stores models, materials and meshes. Up to now, models have a set of associated materials which have a set of meshes. This class hierarchy was set to optimize the draw calls, where we first iterated over models, then over materials and finally over meshes. We will change this structure, not storing meshes under materials any more. Instead, meshes will ve stored directly under the models. We will store materials in a sort of cache, and will have a reference to a key in that cache for the meshes. In addition to that, previously, we created a `Mesh` instance for each of the model meshes, which in essence contained a VAO and the associated VBOs for the mesh data. Since we will be using a single buffer for all the meshes, we will just need a single VAO, ans its associated VBOs, for the whole set of meshes of the scene. Therefore, instead of storing a list of `Mesh`  instances under the `Model` class, we will store the data that will be used to construct the draw parameters, such as the offset on the vertices buffer, the offset for the indices buffer, etc. Let's examine the changes one by one.

We will start with the `MaterialCache` class, which is defined like this:

```java
package org.lwjglb.engine.graph;

import java.util.*;

public class MaterialCache {

    public static final int DEFAULT_MATERIAL_IDX = 0;

    private List<Material> materialsList;

    public MaterialCache() {
        materialsList = new ArrayList<>();
        Material defaultMaterial = new Material();
        materialsList.add(defaultMaterial);
    }

    public void addMaterial(Material material) {
        materialsList.add(material);
        material.setMaterialIdx(materialsList.size() - 1);
    }

    public Material getMaterial(int idx) {
        return materialsList.get(idx);
    }

    public List<Material> getMaterialsList() {
        return materialsList;
    }
}
```

AS you can see, we just store the `Material` instances in a `List`. Therefore, in order to identify a `Material`, we just need the index of that instance in the list. (This approach, may make more difficult to add dynamically new materials, but it is simple enough for the purpose of this sample. You may want to change that, and provide robust support for adding new models, materials, etc. in your code.). We will need to modify the `Material` class to remove the list of `Mesh` instances and store the material index in the materials cache:

```java
public class Material {
    ...
    private Vector4f ambientColor;
    private Vector4f diffuseColor;
    private int materialIdx;
    private String normalMapPath;
    private float reflectance;
    private Vector4f specularColor;
    private String texturePath;

    public Material() {
        diffuseColor = DEFAULT_COLOR;
        ambientColor = DEFAULT_COLOR;
        specularColor = DEFAULT_COLOR;
        materialIdx = 0;
    }
    ...
    public int getMaterialIdx() {
        return materialIdx;
    }
    ...
    public void setMaterialIdx(int materialIdx) {
        this.materialIdx = materialIdx;
    }
    ...
}
```

As it has been explained before, we need to change the `Model` class to remove references to materials. Instead, we will hold two main references:

* A list `MeshData` instances (a new class), which will hold the meshes data read using Assimp.
* A list of `RenderBuffers.MeshDrawData` instances (also a new class), that will contained the information needed for indirect drawing (mainly offsets information associated to the data buffers explained above).

We will first populate the list of `MeshData` instances, when loading the models with assimp, and after that we will construct the global buffers that will hold the data, populating the `RenderBuffers.MeshDrawData` instances. After that, we can remove the references to  `MeshData` instances. This is not a very elegant solution, but it is simple enough to explain the concepts without introducing more complexity using pre and post loading hierarchies. The changes in the `Model` class are as follows:

```java
public class Model {
    ...
    private final String id;
    private List<Animation> animationList;
    private List<Entity> entitiesList;
    private List<MeshData> meshDataList;
    private List<RenderBuffers.MeshDrawData> meshDrawDataList;

    public Model(String id, List<MeshData> meshDataList, List<Animation> animationList) {
        entitiesList = new ArrayList<>();
        this.id = id;
        this.meshDataList = meshDataList;
        this.animationList = animationList;
        meshDrawDataList = new ArrayList<>();
    }
    ...
    public List<MeshData> getMeshDataList() {
        return meshDataList;
    }

    public List<RenderBuffers.MeshDrawData> getMeshDrawDataList() {
        return meshDrawDataList;
    }

    public boolean isAnimated() {
        return animationList != null && !animationList.isEmpty();
    }
    ...
}
```

The definition of the `MeshData` class is very simple. It just stores, vertices positions, texture coordinates, etc:
```java
package org.lwjglb.engine.graph;

import org.joml.Vector3f;

public class MeshData {

    private Vector3f aabbMax;
    private Vector3f aabbMin;
    private float[] bitangents;
    private int[] boneIndices;
    private int[] indices;
    private int materialIdx;
    private float[] normals;
    private float[] positions;
    private float[] tangents;
    private float[] textCoords;
    private float[] weights;

    public MeshData(float[] positions, float[] normals, float[] tangents, float[] bitangents,
                    float[] textCoords, int[] indices, int[] boneIndices, float[] weights,
                    Vector3f aabbMin, Vector3f aabbMax) {
        materialIdx = 0;
        this.positions = positions;
        this.normals = normals;
        this.tangents = tangents;
        this.bitangents = bitangents;
        this.textCoords = textCoords;
        this.indices = indices;
        this.boneIndices = boneIndices;
        this.weights = weights;
        this.aabbMin = aabbMin;
        this.aabbMax = aabbMax;
    }

    public Vector3f getAabbMax() {
        return aabbMax;
    }

    public Vector3f getAabbMin() {
        return aabbMin;
    }

    public float[] getBitangents() {
        return bitangents;
    }

    public int[] getBoneIndices() {
        return boneIndices;
    }

    public int[] getIndices() {
        return indices;
    }

    public int getMaterialIdx() {
        return materialIdx;
    }

    public float[] getNormals() {
        return normals;
    }

    public float[] getPositions() {
        return positions;
    }

    public float[] getTangents() {
        return tangents;
    }

    public float[] getTextCoords() {
        return textCoords;
    }

    public float[] getWeights() {
        return weights;
    }

    public void setMaterialIdx(int materialIdx) {
        this.materialIdx = materialIdx;
    }
}
```

Changes in the `ModelLoader` class are also quite simple, we need to use the materials cache and store the data read in the new `MeshData` class (instead of the previous `Mesh` class). Also, materials wil not have references to mesh data, but mesh data will have a reference to the index of the material in the cache:

```java
public class ModelLoader {
    ...
    public static Model loadModel(String modelId, String modelPath, TextureCache textureCache, MaterialCache materialCache,
                                  boolean animation) {
        return loadModel(modelId, modelPath, textureCache, materialCache, aiProcess_GenSmoothNormals | aiProcess_JoinIdenticalVertices |
                aiProcess_Triangulate | aiProcess_FixInfacingNormals | aiProcess_CalcTangentSpace | aiProcess_LimitBoneWeights |
                aiProcess_GenBoundingBoxes | (animation ? 0 : aiProcess_PreTransformVertices));
    }

    public static Model loadModel(String modelId, String modelPath, TextureCache textureCache,
                                  MaterialCache materialCache, int flags) {
        ...

        for (int i = 0; i < numMaterials; i++) {
            AIMaterial aiMaterial = AIMaterial.create(aiScene.mMaterials().get(i));
            Material material = processMaterial(aiMaterial, modelDir, textureCache);
            materialCache.addMaterial(material);
            materialList.add(material);
        }

        int numMeshes = aiScene.mNumMeshes();
        PointerBuffer aiMeshes = aiScene.mMeshes();
        List<MeshData> meshDataList = new ArrayList<>();
        List<Bone> boneList = new ArrayList<>();
        for (int i = 0; i < numMeshes; i++) {
            AIMesh aiMesh = AIMesh.create(aiMeshes.get(i));
            MeshData meshData = processMesh(aiMesh, boneList);
            int materialIdx = aiMesh.mMaterialIndex();
            if (materialIdx >= 0 && materialIdx < materialList.size()) {
                meshData.setMaterialIdx(materialList.get(materialIdx).getMaterialIdx());
            } else {
                meshData.setMaterialIdx(MaterialCache.DEFAULT_MATERIAL_IDX);
            }
            meshDataList.add(meshData);
        }
        ...
        return new Model(modelId, meshDataList, animations);
    }
    ...
    private static MeshData processMesh(AIMesh aiMesh, List<Bone> boneList) {
        ...
        return new MeshData(vertices, normals, tangents, bitangents, textCoords, indices, animMeshData.boneIds,
                animMeshData.weights, aabbMin, aabbMax);
    }
}
```

The `Scene` class will be the one that will hold the materials cache (also, the `cleanup` mwthod is no longer needed, because the VAOs and VBOs will not be longer linked to the model map):

```java
public class Scene {
    ...
    private MaterialCache materialCache;
    ...
    public Scene(int width, int height) {
        ...
        materialCache = new MaterialCache();
        ...
    }
    ...
    public MaterialCache getMaterialCache() {
        return materialCache;
    }
    ...    
}
```

Changes in the `Mesh` class are due to the fact we introduced the `MeshData` class (just a matter of changing constructor arguments and methods):

```java
public class Mesh {
    ...
    public Mesh(MeshData meshData) {
        try (MemoryStack stack = MemoryStack.stackPush()) {
            this.aabbMin = meshData.getAabbMin();
            this.aabbMax = meshData.getAabbMax();
            numVertices = meshData.getIndices().length;
            ...
            FloatBuffer positionsBuffer = stack.callocFloat(meshData.getPositions().length);
            positionsBuffer.put(0, meshData.getPositions());
            ...
            FloatBuffer normalsBuffer = stack.callocFloat(meshData.getNormals().length);
            normalsBuffer.put(0, meshData.getNormals());
            ...
            FloatBuffer tangentsBuffer = stack.callocFloat(meshData.getTangents().length);
            tangentsBuffer.put(0, meshData.getTangents());
            ...
            FloatBuffer bitangentsBuffer = stack.callocFloat(meshData.getBitangents().length);
            bitangentsBuffer.put(0, meshData.getBitangents());
            ...
            FloatBuffer textCoordsBuffer = MemoryUtil.memAllocFloat(meshData.getTextCoords().length);
            textCoordsBuffer.put(0, meshData.getTextCoords());
            ...
            FloatBuffer weightsBuffer = MemoryUtil.memAllocFloat(meshData.getWeights().length);
            weightsBuffer.put(meshData.getWeights()).flip();
            ...
            IntBuffer boneIndicesBuffer = MemoryUtil.memAllocInt(meshData.getBoneIndices().length);
            boneIndicesBuffer.put(meshData.getBoneIndices()).flip();
            ...
            IntBuffer indicesBuffer = stack.callocInt(meshData.getIndices().length);
            indicesBuffer.put(0, meshData.getIndices());
        }
    }
    ...
}
```

It is turn now for one of the new key classes that we will create for indirect drawing, the `RenderBuffers` class. This class will create a single VAO which will hold the VBOs which will contain the data for all the meshes. In this case, we will just supporting static models, so we will need a single VAO. The `RenderBuffers` class starts like this:

```java
public class RenderBuffers {

    private int staticVaoId;
    private List<Integer> vboIdList;

    public RenderBuffers() {
        vboIdList = new ArrayList<>();
    }

    public void cleanup() {
        vboIdList.stream().forEach(GL30::glDeleteBuffers);
        glDeleteVertexArrays(staticVaoId);
    }
    ...
}
```

This class defines two methods to load models:
* `loadAnimatedModels` for animated models. This will not be implemented in this chapter.
* `loadStaticModels`for models with no animations.

Those methods are defined like this:

```java
public class RenderBuffers {
    ...
    public final int getStaticVaoId() {
        return staticVaoId;
    }

    public void loadAnimatedModels(Scene scene) {
        // To be completed
    }

    public void loadStaticModels(Scene scene) {
        List<Model> modelList = scene.getModelMap().values().stream().filter(m -> !m.isAnimated()).toList();
        staticVaoId = glGenVertexArrays();
        glBindVertexArray(staticVaoId);
        int positionsSize = 0;
        int normalsSize = 0;
        int textureCoordsSize = 0;
        int indicesSize = 0;
        int offset = 0;
        for (Model model : modelList) {
            List<RenderBuffers.MeshDrawData> meshDrawDataList = model.getMeshDrawDataList();
            for (MeshData meshData : model.getMeshDataList()) {
                positionsSize += meshData.getPositions().length;
                normalsSize += meshData.getNormals().length;
                textureCoordsSize += meshData.getTextCoords().length;
                indicesSize += meshData.getIndices().length;

                int meshSizeInBytes = meshData.getPositions().length * 14 * 4;
                meshDrawDataList.add(new MeshDrawData(meshSizeInBytes, meshData.getMaterialIdx(), offset,
                        meshData.getIndices().length));
                offset = positionsSize / 3;
            }
        }

        int vboId = glGenBuffers();
        vboIdList.add(vboId);
        FloatBuffer meshesBuffer = MemoryUtil.memAllocFloat(positionsSize + normalsSize * 3 + textureCoordsSize);
        for (Model model : modelList) {
            for (MeshData meshData : model.getMeshDataList()) {
                populateMeshBuffer(meshesBuffer, meshData);
            }
        }
        meshesBuffer.flip();
        glBindBuffer(GL_ARRAY_BUFFER, vboId);
        glBufferData(GL_ARRAY_BUFFER, meshesBuffer, GL_STATIC_DRAW);
        MemoryUtil.memFree(meshesBuffer);

        defineVertexAttribs();

        // Index VBO
        vboId = glGenBuffers();
        vboIdList.add(vboId);
        IntBuffer indicesBuffer = MemoryUtil.memAllocInt(indicesSize);
        for (Model model : modelList) {
            for (MeshData meshData : model.getMeshDataList()) {
                indicesBuffer.put(meshData.getIndices());
            }
        }
        indicesBuffer.flip();
        glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, vboId);
        glBufferData(GL_ELEMENT_ARRAY_BUFFER, indicesBuffer, GL_STATIC_DRAW);
        MemoryUtil.memFree(indicesBuffer);

        glBindBuffer(GL_ARRAY_BUFFER, 0);
        glBindVertexArray(0);
    }
    ...
}
```

We start by creating a VAO (which will be used for static models), and then iterate over the meshes of the models. We will use a single buffer to hold all the data, so we just iterate over those elements to get the final buffer size. We will calculate the number of position elements, normals, etc. We use that first loop to also populate the offset information that we wil store in a list that will contain `RenderBuffers.MeshDrawData` instances. After that we will create a single VBO. You will find a major difference with the one in the `Mesh` class that did a similar task, creating the VAO and VBOs. In this case, we use a single VBO for positions, normals etc. We just load all that data row by row instead of using separate VBOs. This is done in the `populateMeshBuffer` (which we will see after this). After that, we create the index VBO which will contain the indices for all the meshes of all the models.

The `` is defined like this:

```java
public class RenderBuffers {
    ...
    public record MeshDrawData(int sizeInBytes, int materialIdx, int offset, int vertices) {
    }
}
```

It basically stores the size of the mesh in bytes (`sizeInBytes`), the material index to which it is associated, the offset in the buffer that holds the vertices information and the vertices, the number of indices for this mesh. The offset is measured in "rows" You can  think that the portion of the mesh that holds positions, normals and texture coordinates as a single "row". This "row" holds all the information associated to a single vertex and will processed in teh vertex shader. This is why we just dive by three the number of position elements, each "row" will have three position elements, and the number of "rows" in the positions data will match the number of "Rows" in the normals data and so on.

The `populateMeshBuffer` is defined like this:

```java
public class RenderBuffers {
    ...
    private void populateMeshBuffer(FloatBuffer meshesBuffer, MeshData meshData) {
        float[] positions = meshData.getPositions();
        float[] normals = meshData.getNormals();
        float[] tangents = meshData.getTangents();
        float[] bitangents = meshData.getBitangents();
        float[] textCoords = meshData.getTextCoords();

        int rows = positions.length / 3;
        for (int row = 0; row < rows; row++) {
            int startPos = row * 3;
            int startTextCoord = row * 2;
            meshesBuffer.put(positions[startPos]);
            meshesBuffer.put(positions[startPos + 1]);
            meshesBuffer.put(positions[startPos + 2]);
            meshesBuffer.put(normals[startPos]);
            meshesBuffer.put(normals[startPos + 1]);
            meshesBuffer.put(normals[startPos + 2]);
            meshesBuffer.put(tangents[startPos]);
            meshesBuffer.put(tangents[startPos + 1]);
            meshesBuffer.put(tangents[startPos + 2]);
            meshesBuffer.put(bitangents[startPos]);
            meshesBuffer.put(bitangents[startPos + 1]);
            meshesBuffer.put(bitangents[startPos + 2]);
            meshesBuffer.put(textCoords[startTextCoord]);
            meshesBuffer.put(textCoords[startTextCoord + 1]);
        }
    }
    ...
}
```

As you can see, we just iterate over the "rows" of data and pack positions, normals and texture coordinates into the buffer. The `defineVertexAttribs` is defined like this:

```java
public class RenderBuffers {
    ...
    private void defineVertexAttribs() {
        int stride = 3 * 4 * 4 + 2 * 4;
        int pointer = 0;
        // Positions
        glEnableVertexAttribArray(0);
        glVertexAttribPointer(0, 3, GL_FLOAT, false, stride, pointer);
        pointer += 3 * 4;
        // Normals
        glEnableVertexAttribArray(1);
        glVertexAttribPointer(1, 3, GL_FLOAT, false, stride, pointer);
        pointer += 3 * 4;
        // Tangents
        glEnableVertexAttribArray(2);
        glVertexAttribPointer(2, 3, GL_FLOAT, false, stride, pointer);
        pointer += 3 * 4;
        // Bitangents
        glEnableVertexAttribArray(3);
        glVertexAttribPointer(3, 3, GL_FLOAT, false, stride, pointer);
        pointer += 3 * 4;
        // Texture coordinates
        glEnableVertexAttribArray(4);
        glVertexAttribPointer(4, 2, GL_FLOAT, false, stride, pointer);
    }
    ...
}
```

We just define the vertex attributes for the VAO as in previous examples. The only difference here is that we are using a single VBO for them.


SceneRender
scene.vert
scene.frag
ShadowRender
shadow.vert
SkyBoxRender
SkyBox
TextureCache
Engine
Main

[Next chapter](../chapter-21/chapter-21.md)