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
        vboIdList.forEach(GL30::glDeleteBuffers);
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

The `MeshDrawData` class is defined like this:

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

Prior to examining the changes in the `SceneRender` class, let's start with the vertex shader (`scene.vert`), which starts like this:

```glsl
#version 460

const int MAX_DRAW_ELEMENTS = 100;
const int MAX_ENTITIES = 50;

layout (location=0) in vec3 position;
layout (location=1) in vec3 normal;
layout (location=2) in vec3 tangent;
layout (location=3) in vec3 bitangent;
layout (location=4) in vec2 texCoord;

out vec3 outNormal;
out vec3 outTangent;
out vec3 outBitangent;
out vec2 outTextCoord;
out vec4 outViewPosition;
out vec4 outWorldPosition;
out uint outMaterialIdx;

struct DrawElement
{
    int modelMatrixIdx;
    int materialIdx;
};

uniform mat4 projectionMatrix;
uniform mat4 viewMatrix;
uniform mat4 modelMatrix;
uniform DrawElement drawElements[MAX_DRAW_ELEMENTS];
uniform mat4 modelMatrices[MAX_ENTITIES];
...
```

The first thing that you will notice is that we have increased the version to `460`. We also have removed the constants associated with animations (`MAX_WEIGHTS` and `MAX_BONES`), the attributes for bones indices and the uniform for bone matrices. You will see in next chapter that we will no need this information here for animations. We have created two new constants to define the size oif the `drawElements` and `modelMatrices` uniforms. The `drawElements` uniform will hold `DrawElement` instances. It will have one item per mesh and associated entity. If you remember, we will record a single instruction to draw all the items associated to a mesh, setting the number of instances to be drawn. We will need however, specific per entity data, such as the model matrix. This will be hold in the `drawElements` array, which will also point to the material index to be used. The `modelMatrices` array will just hold the model matrices for each of the entities. Material information will be used in the fragment shader you we pass it using the `outMaterialIdx` output variable.

The `main` function, since we do not have to deal with animations, has been simplified a lot:

```glsl
...
void main()
{
    vec4 initPos = vec4(position, 1.0);
    vec4 initNormal = vec4(normal, 0.0);
    vec4 initTangent = vec4(tangent, 0.0);
    vec4 initBitangent = vec4(bitangent, 0.0);

    uint idx = gl_BaseInstance + gl_InstanceID;
    DrawElement drawElement = drawElements[idx];
    outMaterialIdx = drawElement.materialIdx;
    mat4 modelMatrix =  modelMatrices[drawElement.modelMatrixIdx];
    mat4 modelViewMatrix = viewMatrix * modelMatrix;
    outWorldPosition = modelMatrix * initPos;
    outViewPosition  = viewMatrix * outWorldPosition;
    gl_Position   = projectionMatrix * outViewPosition;
    outNormal     = normalize(modelViewMatrix * initNormal).xyz;
    outTangent    = normalize(modelViewMatrix * initTangent).xyz;
    outBitangent  = normalize(modelViewMatrix * initBitangent).xyz;
    outTextCoord  = texCoord;
}
```

The key here is to get the proper index to access the `drawElements` size. We use the `gl_BaseInstance` and `gl_InstanceID` built-in in variables. When recording the instructions for indirect drawing we will use the `baseInstance` attribute. The value for that attribute will be the one associated to `gl_BaseInstance` built-in in variable. The `gl_InstanceID` will start at `0` whenever we change form a mesh to another, and will be increased for of of the instances of the entities associated to the models. Therefore, by combining this two variables we will be able to access the per-entity specific information in the `drawElements` array.  Once we have the proper index, we just transform positions and normal information as in previous versions of the shader.

The scene fragment shader (`scene.frag`) is defined like this:

```glsl
#version 400

const int MAX_MATERIALS  = 20;
const int MAX_TEXTURES = 16;

in vec3 outNormal;
in vec3 outTangent;
in vec3 outBitangent;
in vec2 outTextCoord;
in vec4 outViewPosition;
in vec4 outWorldPosition;
flat in uint outMaterialIdx;

layout (location = 0) out vec4 buffAlbedo;
layout (location = 1) out vec4 buffNormal;
layout (location = 2) out vec4 buffSpecular;

struct Material
{
    vec4 diffuse;
    vec4 specular;
    float reflectance;
    int normalMapIdx;
    int textureIdx;
};

uniform sampler2D txtSampler[MAX_TEXTURES];
uniform Material materials[MAX_MATERIALS];

vec3 calcNormal(int idx, vec3 normal, vec3 tangent, vec3 bitangent, vec2 textCoords) {
    mat3 TBN = mat3(tangent, bitangent, normal);
    vec3 newNormal = texture(txtSampler[idx], textCoords).rgb;
    newNormal = normalize(newNormal * 2.0 - 1.0);
    newNormal = normalize(TBN * newNormal);
    return newNormal;
}

void main() {
    Material material = materials[outMaterialIdx];
    vec4 text_color = texture(txtSampler[material.textureIdx], outTextCoord);
    vec4 diffuse = text_color + material.diffuse;
    if (diffuse.a < 0.5) {
        discard;
    }
    vec4 specular = text_color + material.specular;

    vec3 normal = outNormal;
    if (material.normalMapIdx > 0) {
        normal = calcNormal(material.normalMapIdx, outNormal, outTangent, outBitangent, outTextCoord);
    }

    buffAlbedo   = vec4(diffuse.xyz, material.reflectance);
    buffNormal   = vec4(0.5 * normal + 0.5, 1.0);
    buffSpecular = specular;
}
```

The main changes are related to the way we access material information and textures. We will now have an array of materials information, which will be accessed by the index we calculated in the vertex shader which is now in the `outMaterialIdx` input variable (which has the `flat` modifier which states that this value should not be interpolated from vertex to fragment stage). We will be using an array of textures to access either regular textures or normal maps. The index to those textures are stored now in the `Material` struct. Since we will be accessing the array of samplers using non constant expressions we need to upgrade GLSL version to 400 (that feature is only available since OpenGL 4.0)


Now it is the turn to examine the changes in the `SceneRender` class. We will start by defining a set of constants that will be used in the code, one handle for the buffer that will have the indirect drawing instructions (`staticRenderBufferHandle`) and the number of drawing commands (`staticDrawCount`). We will need also to modify the `createUniforms` method according to the changes in the shaders shown before:

```java
public class SceneRender {
    ...
    public static final int MAX_DRAW_ELEMENTS = 100;
    public static final int MAX_ENTITIES = 50;
    private static final int COMMAND_SIZE = 5 * 4;
    private static final int MAX_MATERIALS = 20;
    private static final int MAX_TEXTURES = 16;
    ...
    private Map<String, Integer> entitiesIdxMap;
    ...
    private int staticDrawCount;
    private int staticRenderBufferHandle;
    ...
    public SceneRender() {
        ...
        entitiesIdxMap = new HashMap<>();
    }

    private void createUniforms() {
        uniformsMap = new UniformsMap(shaderProgram.getProgramId());
        uniformsMap.createUniform("projectionMatrix");
        uniformsMap.createUniform("viewMatrix");

        for (int i = 0; i < MAX_TEXTURES; i++) {
            uniformsMap.createUniform("txtSampler[" + i + "]");
        }

        for (int i = 0; i < MAX_MATERIALS; i++) {
            String name = "materials[" + i + "]";
            uniformsMap.createUniform(name + ".diffuse");
            uniformsMap.createUniform(name + ".specular");
            uniformsMap.createUniform(name + ".reflectance");
            uniformsMap.createUniform(name + ".normalMapIdx");
            uniformsMap.createUniform(name + ".textureIdx");
        }

        for (int i = 0; i < MAX_DRAW_ELEMENTS; i++) {
            String name = "drawElements[" + i + "]";
            uniformsMap.createUniform(name + ".modelMatrixIdx");
            uniformsMap.createUniform(name + ".materialIdx");
        }

        for (int i = 0; i < MAX_ENTITIES; i++) {
            uniformsMap.createUniform("modelMatrices[" + i + "]");
        }
    }
    ...
}
```

The `entitiesIdxMap` will store the position in the list of entities associated to a model which each entity is located. We store that information in a `Map` using entity identifier as key. We will need this info later on since, the indirect drawing commands will be recorded iterating over meshes associated to each model. The main changes are in the `render` method, which is defined like this:

```java
public class SceneRender {
    ...
    public void render(Scene scene, RenderBuffers renderBuffers, GBuffer gBuffer) {
        glBindFramebuffer(GL_DRAW_FRAMEBUFFER, gBuffer.getGBufferId());
        glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
        glViewport(0, 0, gBuffer.getWidth(), gBuffer.getHeight());
        glDisable(GL_BLEND);

        shaderProgram.bind();

        uniformsMap.setUniform("projectionMatrix", scene.getProjection().getProjMatrix());
        uniformsMap.setUniform("viewMatrix", scene.getCamera().getViewMatrix());

        TextureCache textureCache = scene.getTextureCache();
        List<Texture> textures = textureCache.getAll().stream().toList();
        int numTextures = textures.size();
        if (numTextures > MAX_TEXTURES) {
            Logger.warn("Only " + MAX_TEXTURES + " textures can be used");
        }
        for (int i = 0; i < Math.min(MAX_TEXTURES, numTextures); i++) {
            uniformsMap.setUniform("txtSampler[" + i + "]", i);
            Texture texture = textures.get(i);
            glActiveTexture(GL_TEXTURE0 + i);
            texture.bind();
        }

        int entityIdx = 0;
        for (Model model : scene.getModelMap().values()) {
            List<Entity> entities = model.getEntitiesList();
            for (Entity entity : entities) {
                uniformsMap.setUniform("modelMatrices[" + entityIdx + "]", entity.getModelMatrix());
                entityIdx++;
            }
        }

        // Static meshes
        int drawElement = 0;
        for (Model model: scene.getModelMap().values()) {
            if (model.isAnimated()) {
                continue;
            }
            List<Entity> entities = model.getEntitiesList();
            for (RenderBuffers.MeshDrawData meshDrawData : model.getMeshDrawDataList()) {
                for (Entity entity : entities) {
                    String name = "drawElements[" + drawElement + "]";
                    uniformsMap.setUniform(name + ".modelMatrixIdx", entitiesIdxMap.get(entity.getId()));
                    uniformsMap.setUniform(name + ".materialIdx", meshDrawData.materialIdx());
                    drawElement++;
                }
            }
        }
        glBindBuffer(GL_DRAW_INDIRECT_BUFFER, staticRenderBufferHandle);
        glBindVertexArray(renderBuffers.getStaticVaoId());
        glMultiDrawElementsIndirect(GL_TRIANGLES, GL_UNSIGNED_INT, 0, staticDrawCount, 0);
        glBindVertexArray(0);

        glEnable(GL_BLEND);
        shaderProgram.unbind();
    }
    ...
}
```

You can see that we now have to bind the array of texture samplers and activate all the texture units. In addition to that, we iterate over the entities and set up the uniform values for the model matrices. The next step is to setup the `drawElements` array uniform withe the proper values for each of the entities that will point to the index of the model matrix and the material index. After that, we call the `glMultiDrawElementsIndirect` function to perform the indirect drawing. Prior to that, we need to bind the buffers that hold drawing instructions (drawing commands) and the VAO that holds the meshes and indices data. But, when do we populate the buffer for indirect drawing? The answer is that this not need to be performed each render call, if there are no changes in the number of entities, you can record that buffer once, and use it in each render call. In this specific example, we will just populate that buffer at start-up. This means, that, if you want to make changes in the number of entities, you would nee to re-create that buffer again (you should do that for your own engine).

The method that actually builds the indirect draw buffer is called `setupStaticCommandBuffer` which is defined like this:
```java
public class SceneRender {
    ...
    private void setupStaticCommandBuffer(Scene scene) {
        List<Model> modelList = scene.getModelMap().values().stream().filter(m -> !m.isAnimated()).toList();
        int numMeshes = 0;
        for (Model model : modelList) {
            numMeshes += model.getMeshDrawDataList().size();
        }

        int firstIndex = 0;
        int baseInstance = 0;
        ByteBuffer commandBuffer = MemoryUtil.memAlloc(numMeshes * COMMAND_SIZE);
        for (Model model : modelList) {
            List<Entity> entities = model.getEntitiesList();
            int numEntities = entities.size();
            for (RenderBuffers.MeshDrawData meshDrawData : model.getMeshDrawDataList()) {
                // count
                commandBuffer.putInt(meshDrawData.vertices());
                // instanceCount
                commandBuffer.putInt(numEntities);
                commandBuffer.putInt(firstIndex);
                // baseVertex
                commandBuffer.putInt(meshDrawData.offset());
                commandBuffer.putInt(baseInstance);

                firstIndex += meshDrawData.vertices();
                baseInstance += entities.size();
            }
        }
        commandBuffer.flip();

        staticDrawCount = commandBuffer.remaining() / COMMAND_SIZE;

        staticRenderBufferHandle = glGenBuffers();
        glBindBuffer(GL_DRAW_INDIRECT_BUFFER, staticRenderBufferHandle);
        glBufferData(GL_DRAW_INDIRECT_BUFFER, commandBuffer, GL_DYNAMIC_DRAW);

        MemoryUtil.memFree(commandBuffer);
    }
    ...
}
```

We first calculate the total number of meshes. After that, we will create the buffer that wil hold indirect drawing instructions and populate it. As you can see we first allocate a `ByteBuffer`. This buffer will hold as many instruction sets as meshes. Each set of draw instructions si composed by five attributes, each of them with a length of 4 bytes (total length of each set of parameters is what defines the `COMMAND_SIZE` constant). We cannot allocate this buffer using `MemoryStack` since we will run out of space quickly (the stack that LWJGL uses for this is limited in size). Therefore, we need to allocate it using `MemoryUtil` and remember to manually de-allocate that once we are done. Once we have the buffer we start iterating over the meshes associated to the model. You may have a look at the beginning of this chapter to check
 the struct that draw indirect requires. In addition to that, we also populate the `drawElements` uniform using the `Map` we calculated previously, to properly get the model matrix index for each entity. Finally, we just create a GPU buffer and dump the data into it.

We will need to update the `cleanup` method to free the indirect drawing buffer:

```java
public class SceneRender {
    ...
    public void cleanup() {
        shaderProgram.cleanup();
        glDeleteBuffers(staticRenderBufferHandle);
    }
    ...
}
```

We will need a new method to the set up the values for the materials uniform:

```java
public class SceneRender {
    ...
    private void setupMaterialsUniform(TextureCache textureCache, MaterialCache materialCache) {
        List<Texture> textures = textureCache.getAll().stream().toList();
        int numTextures = textures.size();
        if (numTextures > MAX_TEXTURES) {
            Logger.warn("Only " + MAX_TEXTURES + " textures can be used");
        }
        Map<String, Integer> texturePosMap = new HashMap<>();
        for (int i = 0; i < Math.min(MAX_TEXTURES, numTextures); i++) {
            texturePosMap.put(textures.get(i).getTexturePath(), i);
        }

        shaderProgram.bind();
        List<Material> materialList = materialCache.getMaterialsList();
        int numMaterials = materialList.size();
        for (int i = 0; i < numMaterials; i++) {
            Material material = materialCache.getMaterial(i);
            String name = "materials[" + i + "]";
            uniformsMap.setUniform(name + ".diffuse", material.getDiffuseColor());
            uniformsMap.setUniform(name + ".specular", material.getSpecularColor());
            uniformsMap.setUniform(name + ".reflectance", material.getReflectance());
            String normalMapPath = material.getNormalMapPath();
            int idx = 0;
            if (normalMapPath != null) {
                idx = texturePosMap.computeIfAbsent(normalMapPath, k -> 0);
            }
            uniformsMap.setUniform(name + ".normalMapIdx", idx);
            Texture texture = textureCache.getTexture(material.getTexturePath());
            idx = texturePosMap.computeIfAbsent(texture.getTexturePath(), k -> 0);
            uniformsMap.setUniform(name + ".textureIdx", idx);
        }
        shaderProgram.unbind();
    }
    ...
}
```

We just check that we are not surpassing the maximum number of supported textures (`MAX_TEXTURES`)  and just create an array of materials information with the information we used in the previous chapters. The only change is that we will need to store the index of the associated texture and normal maps in the material information.

We need another method to update the entities indices map:
```java
public class SceneRender {
    ...
    private void setupEntitiesData(Scene scene) {
        entitiesIdxMap.clear();
        int entityIdx = 0;
        for (Model model : scene.getModelMap().values()) {
            List<Entity> entities = model.getEntitiesList();
            for (Entity entity : entities) {
                entitiesIdxMap.put(entity.getId(), entityIdx);
                entityIdx++;
            }
        }
    }
    ...
}
```

To complete the changes in the `SceneRender` class, we will create a method that wraps the `setupXX` so it can be invoked from the `Render` class:

```java
public class SceneRender {
    ...
    public void setupData(Scene scene) {
        setupEntitiesData(scene);
        setupStaticCommandBuffer(scene);
        setupMaterialsUniform(scene.getTextureCache(), scene.getMaterialCache());
    }
    ...
}
```

We will change also the shadow render process to use indirect drawing. The changes in the vertex shader (`shadow.vert`) are quite similar, we will not be using animation information and we need to access the proper model matrices using the combination of `gl_BaseInstance` and `gl_InstanceID` built-in variables. In this case, we do not need material information so the fragment shader (`shadow.frag`) is not changed.

```glsl
#version 460

const int MAX_DRAW_ELEMENTS = 100;
const int MAX_ENTITIES = 50;

layout (location=0) in vec3 position;
layout (location=1) in vec3 normal;
layout (location=2) in vec3 tangent;
layout (location=3) in vec3 bitangent;
layout (location=4) in vec2 texCoord;

struct DrawElement
{
    int modelMatrixIdx;
};

uniform mat4 modelMatrix;
uniform mat4 projViewMatrix;
uniform DrawElement drawElements[MAX_DRAW_ELEMENTS];
uniform mat4 modelMatrices[MAX_ENTITIES];

void main()
{
    vec4 initPos = vec4(position, 1.0);
    uint idx = gl_BaseInstance + gl_InstanceID;
    int modelMatrixIdx = drawElements[idx].modelMatrixIdx;
    mat4 modelMatrix = modelMatrices[modelMatrixIdx];
    gl_Position = projViewMatrix * modelMatrix * initPos;
}
```

Changes in `ShadowRender` are also pretty similar as the ones in the `SceneRender` class:

```java
public class ShadowRender {

    private static final int COMMAND_SIZE = 5 * 4;
    ...
    private Map<String, Integer> entitiesIdxMap;
    ...
    private int staticRenderBufferHandle;
    ...
    public ShadowRender() {
        ...
        entitiesIdxMap = new HashMap<>();
    }

    public void cleanup() {
        shaderProgram.cleanup();
        shadowBuffer.cleanup();
        glDeleteBuffers(staticRenderBufferHandle);
    }

    private void createUniforms() {
        ...
        for (int i = 0; i < SceneRender.MAX_DRAW_ELEMENTS; i++) {
            String name = "drawElements[" + i + "]";
            uniformsMap.createUniform(name + ".modelMatrixIdx");
        }

        for (int i = 0; i < SceneRender.MAX_ENTITIES; i++) {
            uniformsMap.createUniform("modelMatrices[" + i + "]");
        }
    }
    ...
}
```

The `createUniforms` method needs to be update to use the new uniforms and the `cleanup` one needs to free the indirect draw buffer. The `render` method will use now the `glMultiDrawElementsIndirect`instead of submitting individual draw commands for meshes and entities:

```java
public class ShadowRender {
    ...
    public void render(Scene scene, RenderBuffers renderBuffers) {
        CascadeShadow.updateCascadeShadows(cascadeShadows, scene);

        glBindFramebuffer(GL_FRAMEBUFFER, shadowBuffer.getDepthMapFBO());
        glViewport(0, 0, ShadowBuffer.SHADOW_MAP_WIDTH, ShadowBuffer.SHADOW_MAP_HEIGHT);

        shaderProgram.bind();

        int entityIdx = 0;
        for (Model model : scene.getModelMap().values()) {
            List<Entity> entities = model.getEntitiesList();
            for (Entity entity : entities) {
                uniformsMap.setUniform("modelMatrices[" + entityIdx + "]", entity.getModelMatrix());
                entityIdx++;
            }
        }

        for (int i = 0; i < CascadeShadow.SHADOW_MAP_CASCADE_COUNT; i++) {
            glFramebufferTexture2D(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT, GL_TEXTURE_2D, shadowBuffer.getDepthMapTexture().getIds()[i], 0);
            glClear(GL_DEPTH_BUFFER_BIT);
        }

        // Static meshes
        int drawElement = 0;
        for (Model model: scene.getModelMap().values()) {
            if (model.isAnimated()) {
                continue;
            }
            List<Entity> entities = model.getEntitiesList();
            for (RenderBuffers.MeshDrawData meshDrawData : model.getMeshDrawDataList()) {
                for (Entity entity : entities) {
                    String name = "drawElements[" + drawElement + "]";
                    uniformsMap.setUniform(name + ".modelMatrixIdx", entitiesIdxMap.get(entity.getId()));
                    drawElement++;
                }
            }
        }
        glBindBuffer(GL_DRAW_INDIRECT_BUFFER, staticRenderBufferHandle);
        glBindVertexArray(renderBuffers.getStaticVaoId());
        for (int i = 0; i < CascadeShadow.SHADOW_MAP_CASCADE_COUNT; i++) {
            glFramebufferTexture2D(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT, GL_TEXTURE_2D, shadowBuffer.getDepthMapTexture().getIds()[i], 0);

            CascadeShadow shadowCascade = cascadeShadows.get(i);
            uniformsMap.setUniform("projViewMatrix", shadowCascade.getProjViewMatrix());

            glMultiDrawElementsIndirect(GL_TRIANGLES, GL_UNSIGNED_INT, 0, staticDrawCount, 0);
        }
        glBindVertexArray(0);

        shaderProgram.unbind();
        glBindFramebuffer(GL_FRAMEBUFFER, 0);
    }
    ...
}
```
Finally, we need a similar method to set up the indirect draw buffer and the entities map:

```java
public class ShadowRender {
    ...
    public void setupData(Scene scene) {
        setupEntitiesData(scene);
        setupStaticCommandBuffer(scene);
    }

    private void setupEntitiesData(Scene scene) {
        entitiesIdxMap.clear();
        int entityIdx = 0;
        for (Model model : scene.getModelMap().values()) {
            List<Entity> entities = model.getEntitiesList();
            for (Entity entity : entities) {
                entitiesIdxMap.put(entity.getId(), entityIdx);
                entityIdx++;
            }
        }
    }

    private void setupStaticCommandBuffer(Scene scene) {
        List<Model> modelList = scene.getModelMap().values().stream().filter(m -> !m.isAnimated()).toList();
        Map<String, Integer> entitiesIdxMap = new HashMap<>();
        int entityIdx = 0;
        int numMeshes = 0;
        for (Model model : scene.getModelMap().values()) {
            List<Entity> entities = model.getEntitiesList();
            numMeshes += model.getMeshDrawDataList().size();
            for (Entity entity : entities) {
                entitiesIdxMap.put(entity.getId(), entityIdx);
                entityIdx++;
            }
        }

        int firstIndex = 0;
        int baseInstance = 0;
        int drawElement = 0;
        shaderProgram.bind();
        ByteBuffer commandBuffer = MemoryUtil.memAlloc(numMeshes * COMMAND_SIZE);
        for (Model model : modelList) {
            List<Entity> entities = model.getEntitiesList();
            int numEntities = entities.size();
            for (RenderBuffers.MeshDrawData meshDrawData : model.getMeshDrawDataList()) {
                // count
                commandBuffer.putInt(meshDrawData.vertices());
                // instanceCount
                commandBuffer.putInt(numEntities);
                commandBuffer.putInt(firstIndex);
                // baseVertex
                commandBuffer.putInt(meshDrawData.offset());
                commandBuffer.putInt(baseInstance);

                firstIndex += meshDrawData.vertices();
                baseInstance += entities.size();

                for (Entity entity : entities) {
                    String name = "drawElements[" + drawElement + "]";
                    uniformsMap.setUniform(name + ".modelMatrixIdx", entitiesIdxMap.get(entity.getId()));
                    drawElement++;
                }
            }
        }
        commandBuffer.flip();
        shaderProgram.unbind();

        staticDrawCount = commandBuffer.remaining() / COMMAND_SIZE;

        staticRenderBufferHandle = glGenBuffers();
        glBindBuffer(GL_DRAW_INDIRECT_BUFFER, staticRenderBufferHandle);
        glBufferData(GL_DRAW_INDIRECT_BUFFER, commandBuffer, GL_DYNAMIC_DRAW);

        MemoryUtil.memFree(commandBuffer);
    }
}
```

In the `Render` class, we just need to instantiate the `RenderBuffers` class and provide a new method `setupData` which can be called when every model and entity has been created to create the indirect drawing buffers and associated data.

```java
public class Render {
    ...
    private RenderBuffers renderBuffers;
    ...
    public Render(Window window) {
        ...
        renderBuffers = new RenderBuffers();
    }

    public void cleanup() {
        ...
        renderBuffers.cleanup();
    }
    ...
    public void render(Window window, Scene scene) {
        shadowRender.render(scene, renderBuffers);
        sceneRender.render(scene, renderBuffers, gBuffer);
        ...
    }
    ...
    public void setupData(Scene scene) {
        renderBuffers.loadStaticModels(scene);
        renderBuffers.loadAnimatedModels(scene);
        sceneRender.setupData(scene);
        shadowRender.setupData(scene);
        List<Model> modelList = new ArrayList<>(scene.getModelMap().values());
        modelList.forEach(m -> m.getMeshDataList().clear());
    }
}
```

We need toi update the `TextureCache` class to provide a method to return all the textures:

```java
public class TextureCache {
    ...
    public Collection<Texture> getAll() {
        return textureMap.values();
    }
    ...
}
```

Since we have modified the class hierarchy that deals with models and materials, we need to update the `SkyBox` class (loading individual models require now additional steps):

```java
public class SkyBox {

    private Material material;
    private Mesh mesh;
    ...
    public SkyBox(String skyBoxModelPath, TextureCache textureCache, MaterialCache materialCache) {
        skyBoxModel = ModelLoader.loadModel("skybox-model", skyBoxModelPath, textureCache, materialCache, false);
        MeshData meshData = skyBoxModel.getMeshDataList().get(0);
        material = materialCache.getMaterial(meshData.getMaterialIdx());
        mesh = new Mesh(meshData);
        skyBoxModel.getMeshDataList().clear();
        skyBoxEntity = new Entity("skyBoxEntity-entity", skyBoxModel.getId());
    }

    public void cleanuo() {
        mesh.cleanup();
    }

    public Material getMaterial() {
        return material;
    }

    public Mesh getMesh() {
        return mesh;
    }
    ...
}
```

These changes also affect the `SkyBoxRender` class. For sky bos render we will not use indirect drawing (it is not worth it since we will be rendering just one mesh):

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

        Entity skyBoxEntity = skyBox.getSkyBoxEntity();
        TextureCache textureCache = scene.getTextureCache();
        Material material = skyBox.getMaterial();
        Mesh mesh = skyBox.getMesh();
        Texture texture = textureCache.getTexture(material.getTexturePath());
        glActiveTexture(GL_TEXTURE0);
        texture.bind();

        uniformsMap.setUniform("diffuse", material.getDiffuseColor());
        uniformsMap.setUniform("hasTexture", texture.getTexturePath().equals(TextureCache.DEFAULT_TEXTURE) ? 0 : 1);

        glBindVertexArray(mesh.getVaoId());

        uniformsMap.setUniform("modelMatrix", skyBoxEntity.getModelMatrix());
        glDrawElements(GL_TRIANGLES, mesh.getNumVertices(), GL_UNSIGNED_INT, 0);

        glBindVertexArray(0);

        shaderProgram.unbind();
    }
    ...
}
```

In the `Scene` class, we just do not need to invoke the `Scene` `cleanup` method (since the data associated to the buffers is in the `RenderBuffers` class):

```java
public class Engine {
    ...
    private void cleanup() {
        appLogic.cleanup();
        render.cleanup();
        window.cleanup();
    }
    ...
}
```

Finally, in the `Main` class, we will load two entities associated to a cube model. We will rotate them independently to check that the code works ok. The most important part is to cal the `Render` class `setupData` method when everything is loaded.

```java
public class Main implements IAppLogic {
    ...
    private Entity cubeEntity1;
    private Entity cubeEntity2;
    ...
    private float rotation;

    public static void main(String[] args) {
        ...
        Engine gameEng = new Engine("chapter-20", opts, main);
        ...
    }

    public void init(Window window, Scene scene, Render render) {
        ...
        Model terrainModel = ModelLoader.loadModel(terrainModelId, "resources/models/terrain/terrain.obj",
                scene.getTextureCache(), scene.getMaterialCache(), false);
        ...
        Model cubeModel = ModelLoader.loadModel("cube-model", "resources/models/cube/cube.obj",
                scene.getTextureCache(), scene.getMaterialCache(), false);
        scene.addModel(cubeModel);
        cubeEntity1 = new Entity("cube-entity-1", cubeModel.getId());
        cubeEntity1.setPosition(0, 2, -1);
        cubeEntity1.updateModelMatrix();
        scene.addEntity(cubeEntity1);

        cubeEntity2 = new Entity("cube-entity-2", cubeModel.getId());
        cubeEntity2.setPosition(-2, 2, -1);
        cubeEntity2.updateModelMatrix();
        scene.addEntity(cubeEntity2);

        render.setupData(scene);
        ...
        SkyBox skyBox = new SkyBox("resources/models/skybox/skybox.obj", scene.getTextureCache(),
                scene.getMaterialCache());
        ...
    }
    ...
    public void update(Window window, Scene scene, long diffTimeMillis) {
        rotation += 1.5;
        if (rotation > 360) {
            rotation = 0;
        }
        cubeEntity1.setRotation(1, 1, 1, (float) Math.toRadians(rotation));
        cubeEntity1.updateModelMatrix();

        cubeEntity2.setRotation(1, 1, 1, (float) Math.toRadians(360 - rotation));
        cubeEntity2.updateModelMatrix();
    }
}

```

With all of that changes implemented you should be able to see something similar to this.

![Screen shot](screenshot.png)


[Next chapter](../chapter-21/chapter-21.md)