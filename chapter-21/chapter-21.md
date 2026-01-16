# Chapter 21 - Indirect drawing (animated models) and compute shaders

In this chapter we will add support for animated models when using indirect drawing. In order to do so, we will introduce a new topic, compute shaders. We will use compute shaders to transform model vertices from the binding pose to their final position (according to current animation). Once we have done this, we can use regular shaders to render them, there will be no need to distinguish between animated and non animated models while rendering. In addition to that, we will be able decouple animation transformations from the rendering process. By doing so, we will be able to update animation models in a different rate than the render rate (we do not need to transform animated vertices in each frame if they have not changed).

You can find the complete source code for this chapter [here](https://github.com/lwjglgamedev/lwjglbook/tree/main/chapter-20).

## Concepts

Prior to explaining the code, let's explain the concepts behind indirect drawing for animated models. The approach we will follow will be more or less the same as the one used in the previous chapter. We will have a global buffer which will contain vertices data. The main difference is that we will use first a compute shader to transform vertices from the binding pose to the final one. In addition to that, we will not use multiple instances for a model. The reason for that is, even if we have several entities that share the same animated model, they can be in different animation state (the animation may have started after, have a lower update rate or even the specific selected animation of the model may be different). Therefore we will need, inside the global buffer that will contain the animated vertices, a single chunk of data per entity.

We will still need to keep binding data, we will create another global buffer for that for all the meshes of the scene. In this case we do not need to have separate chunks per entity, just one per mesh. The compute shader will access that binding poses data buffer, will process that for each of the entities and will store the results into another global buffer with a structure similar to the one used for static models.

## Model loading

We need to update the `Model` class since we will not store bone matrices data any more in this class. Instead, that information will be stored in a common buffer. Therefore the inner class `AnimatedFrame` cannot be a record any longer (records are immutable).

```java
public class Model {
    ...
    public static class AnimatedFrame {
        private Matrix4f[] bonesMatrices;
        private int offset;

        public AnimatedFrame(Matrix4f[] bonesMatrices) {
            this.bonesMatrices = bonesMatrices;
        }

        public void clearData() {
            bonesMatrices = null;
        }

        public Matrix4f[] getBonesMatrices() {
            return bonesMatrices;
        }

        public int getOffset() {
            return offset;
        }

        public void setOffset(int offset) {
            this.offset = offset;
        }
    }
    ...
}
```

The fact that we pass from a record to a regular inner class, changing the way we access `Model` class attributes requires a slight modification in the `ModelLoader` class:

```java
public class ModelLoader {
    ...
    private static void buildFrameMatrices(AIAnimation aiAnimation, List<Bone> boneList, Model.AnimatedFrame animatedFrame,
                                           int frame, Node node, Matrix4f parentTransformation, Matrix4f globalInverseTransform) {
        ...
        for (Bone bone : affectedBones) {
            ...
            animatedFrame.getBonesMatrices()[bone.boneId()] = boneTransform;
        }
        ...
    }
    ...
}
```

Let's review now the new global buffers that we will need, which will be managed in the `RenderBuffers` class:

```java
public class RenderBuffers {

    private int animVaoId;
    private int bindingPosesBuffer;
    private int bonesIndicesWeightsBuffer;
    private int bonesMatricesBuffer;
    private int destAnimationBuffer;
    ...
    public void cleanup() {
        ...
        glDeleteVertexArrays(animVaoId);
    }
    ...
    public int getAnimVaoId() {
        return animVaoId;
    }

    public int getBindingPosesBuffer() {
        return bindingPosesBuffer;
    }

    public int getBonesIndicesWeightsBuffer() {
        return bonesIndicesWeightsBuffer;
    }

    public int getBonesMatricesBuffer() {
        return bonesMatricesBuffer;
    }

    public int getDestAnimationBuffer() {
        return destAnimationBuffer;
    }
    ...
}
```

The `animVaoId` will store the VAO which will define the data which will contain the transformed animation vertices, that is, the data after it has been processed by the compute shader (remember one chunk per mesh and entity). The data itself will be stored in a buffer, whose handle will be stored in `destAnimationBuffer`. We need to access that buffer in the compute shader which does not understand VAOs, just buffers. We will need also to store bone matrices and indices and weights into two buffers represented by `bonesMatricesBuffer` and `bonesIndicesWeightsBuffer` respectively. In the `cleanup` method we must not forget to clean the new VAO. We also need to add getters for the new attributes.

We can now implement the `loadAnimatedModels` which starts like this:

```java
public class RenderBuffers {
    ...
    public void loadAnimatedModels(Scene scene) {
        List<Model> modelList = scene.getModelMap().values().stream().filter(Model::isAnimated).toList();
        loadBindingPoses(modelList);
        loadBonesMatricesBuffer(modelList);
        loadBonesIndicesWeights(modelList);

        animVaoId = glGenVertexArrays();
        glBindVertexArray(animVaoId);
        int positionsSize = 0;
        int normalsSize = 0;
        int textureCoordsSize = 0;
        int indicesSize = 0;
        int offset = 0;
        int chunkBindingPoseOffset = 0;
        int bindingPoseOffset = 0;
        int chunkWeightsOffset = 0;
        int weightsOffset = 0;
        for (Model model : modelList) {
            List<Entity> entities = model.getEntitiesList();
            for (Entity entity : entities) {
                List<RenderBuffers.MeshDrawData> meshDrawDataList = model.getMeshDrawDataList();
                bindingPoseOffset = chunkBindingPoseOffset;
                weightsOffset = chunkWeightsOffset;
                for (MeshData meshData : model.getMeshDataList()) {
                    positionsSize += meshData.getPositions().length;
                    normalsSize += meshData.getNormals().length;
                    textureCoordsSize += meshData.getTextCoords().length;
                    indicesSize += meshData.getIndices().length;

                    int meshSizeInBytes = (meshData.getPositions().length + meshData.getNormals().length * 3 + meshData.getTextCoords().length) * 4;
                    meshDrawDataList.add(new MeshDrawData(meshSizeInBytes, meshData.getMaterialIdx(), offset,
                            meshData.getIndices().length, new AnimMeshDrawData(entity, bindingPoseOffset, weightsOffset)));
                    bindingPoseOffset += meshSizeInBytes / 4;
                    int groupSize = (int) Math.ceil((float) meshSizeInBytes / (14 * 4));
                    weightsOffset += groupSize * 2 * 4;
                    offset = positionsSize / 3;
                }
            }
            chunkBindingPoseOffset += bindingPoseOffset;
            chunkWeightsOffset += weightsOffset;
        }

        destAnimationBuffer = glGenBuffers();
        vboIdList.add(destAnimationBuffer);
        FloatBuffer meshesBuffer = MemoryUtil.memAllocFloat(positionsSize + normalsSize * 3 + textureCoordsSize);
        for (Model model : modelList) {
            model.getEntitiesList().forEach(e -> {
                for (MeshData meshData : model.getMeshDataList()) {
                    populateMeshBuffer(meshesBuffer, meshData);
                }
            });
        }
        meshesBuffer.flip();
        glBindBuffer(GL_ARRAY_BUFFER, destAnimationBuffer);
        glBufferData(GL_ARRAY_BUFFER, meshesBuffer, GL_STATIC_DRAW);
        MemoryUtil.memFree(meshesBuffer);

        defineVertexAttribs();

        // Index VBO
        int vboId = glGenBuffers();
        vboIdList.add(vboId);
        IntBuffer indicesBuffer = MemoryUtil.memAllocInt(indicesSize);
        for (Model model : modelList) {
            model.getEntitiesList().forEach(e -> {
                for (MeshData meshData : model.getMeshDataList()) {
                    indicesBuffer.put(meshData.getIndices());
                }
            });
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

We will see later on how the following methods are defined but, by now:

* `loadBindingPoses`: Stores binding pose information for all the meshes associated to animated model.
* `loadBonesMatricesBuffer` : Stores the bone matrices for each animation of the animated models.
* `loadBonesIndicesWeights`: Stores the bones indices and weights information of the animated models.

The code is very similar to the `loadStaticModels`, we start by creating a VAO for animated models, and then iterate over the meshes of the models. We will use a single buffer to hold all the data, so we just iterate over those elements to get the final buffer size. Please note that the first loop is a little bit different than the static version. We need to iterate over the entities associated to a model, and for each of them we calculate the size of all the associated meshes.

Let's examine the `loadBindingPoses` method:

```java
public class RenderBuffers {
    ...
    private void loadBindingPoses(List<Model> modelList) {
        int meshSize = 0;
        for (Model model : modelList) {
            for (MeshData meshData : model.getMeshDataList()) {
                meshSize += meshData.getPositions().length + meshData.getNormals().length * 3 +
                        meshData.getTextCoords().length + meshData.getIndices().length;
            }
        }

        bindingPosesBuffer = glGenBuffers();
        vboIdList.add(bindingPosesBuffer);
        FloatBuffer meshesBuffer = MemoryUtil.memAllocFloat(meshSize);
        for (Model model : modelList) {
            for (MeshData meshData : model.getMeshDataList()) {
                populateMeshBuffer(meshesBuffer, meshData);
            }
        }
        meshesBuffer.flip();
        glBindBuffer(GL_SHADER_STORAGE_BUFFER, bindingPosesBuffer);
        glBufferData(GL_SHADER_STORAGE_BUFFER, meshesBuffer, GL_STATIC_DRAW);
        MemoryUtil.memFree(meshesBuffer);

        glBindBuffer(GL_ARRAY_BUFFER, 0);
    }
    ...
}
```

The `loadBindingPoses` iterates over all the animated models, getting the total size to accommodate all the associated meshes. With that size, a buffer is created and populated using the `populateMeshBuffer` which was already present in the chapter before. Therefore, we store binding pose vertices for all the meshes of the animated models into a single buffer. We will access this buffer in the compute shader, so you can see that we use the `GL_SHADER_STORAGE_BUFFER` flag when binding.

The `loadBonesMatricesBuffer` method is defined like this:

```java
public class RenderBuffers {
    ...
    private void loadBonesMatricesBuffer(List<Model> modelList) {
        int bufferSize = 0;
        for (Model model : modelList) {
            List<Model.Animation> animationsList = model.getAnimationList();
            for (Model.Animation animation : animationsList) {
                List<Model.AnimatedFrame> frameList = animation.frames();
                for (Model.AnimatedFrame frame : frameList) {
                    Matrix4f[] matrices = frame.getBonesMatrices();
                    bufferSize += matrices.length * 64;
                }
            }
        }

        bonesMatricesBuffer = glGenBuffers();
        vboIdList.add(bonesMatricesBuffer);
        ByteBuffer dataBuffer = MemoryUtil.memAlloc(bufferSize);
        int matrixSize = 4 * 4 * 4;
        for (Model model : modelList) {
            List<Model.Animation> animationsList = model.getAnimationList();
            for (Model.Animation animation : animationsList) {
                List<Model.AnimatedFrame> frameList = animation.frames();
                for (Model.AnimatedFrame frame : frameList) {
                    frame.setOffset(dataBuffer.position() / matrixSize);
                    Matrix4f[] matrices = frame.getBonesMatrices();
                    for (Matrix4f matrix : matrices) {
                        matrix.get(dataBuffer);
                        dataBuffer.position(dataBuffer.position() + matrixSize);
                    }
                    frame.clearData();
                }
            }
        }
        dataBuffer.flip();

        glBindBuffer(GL_SHADER_STORAGE_BUFFER, bonesMatricesBuffer);
        glBufferData(GL_SHADER_STORAGE_BUFFER, dataBuffer, GL_STATIC_DRAW);
        MemoryUtil.memFree(dataBuffer);
    }
    ...
}
```

We start iterating over the animation data for each of the models, getting the associated transformation matrices (for all the bones) for each of the animated frames in order to calculate the buffer that will hold all that information. Once we have the size, we create the buffer and start populating that (in the second loop) with those matrices. As in the previous buffer we will access this buffer in the compute shader, therefore we need to use the `GL_SHADER_STORAGE_BUFFER` flag.

The `loadBonesIndicesWeights` method is defined like this:

```java
public class RenderBuffers {
    ...
    private void loadBonesIndicesWeights(List<Model> modelList) {
        int bufferSize = 0;
        for (Model model : modelList) {
            for (MeshData meshData : model.getMeshDataList()) {
                bufferSize += meshData.getBoneIndices().length * 4 + meshData.getWeights().length * 4;
            }
        }
        ByteBuffer dataBuffer = MemoryUtil.memAlloc(bufferSize);
        for (Model model : modelList) {
            for (MeshData meshData : model.getMeshDataList()) {
                int[] bonesIndices = meshData.getBoneIndices();
                float[] weights = meshData.getWeights();
                int rows = bonesIndices.length / 4;
                for (int row = 0; row < rows; row++) {
                    int startPos = row * 4;
                    dataBuffer.putFloat(weights[startPos]);
                    dataBuffer.putFloat(weights[startPos + 1]);
                    dataBuffer.putFloat(weights[startPos + 2]);
                    dataBuffer.putFloat(weights[startPos + 3]);
                    dataBuffer.putFloat(bonesIndices[startPos]);
                    dataBuffer.putFloat(bonesIndices[startPos + 1]);
                    dataBuffer.putFloat(bonesIndices[startPos + 2]);
                    dataBuffer.putFloat(bonesIndices[startPos + 3]);
                }
            }
        }
        dataBuffer.flip();

        bonesIndicesWeightsBuffer = glGenBuffers();
        vboIdList.add(bonesIndicesWeightsBuffer);
        glBindBuffer(GL_SHADER_STORAGE_BUFFER, bonesIndicesWeightsBuffer);
        glBufferData(GL_SHADER_STORAGE_BUFFER, dataBuffer, GL_STATIC_DRAW);
        MemoryUtil.memFree(dataBuffer);

        glBindBuffer(GL_SHADER_STORAGE_BUFFER, 0);
    }
    ...
}
```

As in the previous methods, we will store the weights and bone indices information into a single buffer, so we need to first calculate its size and later on populate it. As in the previous buffer we will access this buffers in the compute shader, therefore we need to use the `GL_SHADER_STORAGE_BUFFER` flag.

## Compute shaders

It is turn now to implement animation transformations through compute shaders. As it has been said before, a shader is like any other shader but it does not compose any restrictions on its inputs and its outputs. We will use them to transform data, they will have access to the global buffers that hold information about binding poses and animation transformation matrices and it will dump the result into another buffer. The shader code for animations (`anim.comp`) is defined like this:

```glsl
#version 460

layout (std430, binding=0) readonly buffer srcBuf {
    float data[];
} srcVector;

layout (std430, binding=1) readonly buffer weightsBuf {
    float data[];
} weightsVector;

layout (std430, binding=2) readonly buffer bonesBuf {
    mat4 data[];
} bonesMatrices;

layout (std430, binding=3) buffer dstBuf {
    float data[];
} dstVector;

struct DrawParameters
{
    int srcOffset;
    int srcSize;
    int weightsOffset;
    int bonesMatricesOffset;
    int dstOffset;
};
uniform DrawParameters drawParameters;

layout (local_size_x=1, local_size_y=1, local_size_z=1) in;

void main()
{
    int baseIdx = int(gl_GlobalInvocationID.x) * 14;
    uint baseIdxWeightsBuf  = drawParameters.weightsOffset + int(gl_GlobalInvocationID.x) * 8;
    uint baseIdxSrcBuf = drawParameters.srcOffset + baseIdx;
    uint baseIdxDstBuf = drawParameters.dstOffset + baseIdx;
    if (baseIdx >= drawParameters.srcSize) {
        return;
    }

    vec4 weights = vec4(weightsVector.data[baseIdxWeightsBuf], weightsVector.data[baseIdxWeightsBuf + 1], weightsVector.data[baseIdxWeightsBuf + 2], weightsVector.data[baseIdxWeightsBuf + 3]);
    ivec4 bonesIndices = ivec4(weightsVector.data[baseIdxWeightsBuf + 4], weightsVector.data[baseIdxWeightsBuf + 5], weightsVector.data[baseIdxWeightsBuf + 6], weightsVector.data[baseIdxWeightsBuf + 7]);

    vec4 position = vec4(srcVector.data[baseIdxSrcBuf], srcVector.data[baseIdxSrcBuf + 1], srcVector.data[baseIdxSrcBuf + 2], 1);
    position =
    weights.x * bonesMatrices.data[drawParameters.bonesMatricesOffset + bonesIndices.x] * position +
    weights.y * bonesMatrices.data[drawParameters.bonesMatricesOffset + bonesIndices.y] * position +
    weights.z * bonesMatrices.data[drawParameters.bonesMatricesOffset + bonesIndices.z] * position +
    weights.w * bonesMatrices.data[drawParameters.bonesMatricesOffset + bonesIndices.w] * position;
    dstVector.data[baseIdxDstBuf] = position.x / position.w;
    dstVector.data[baseIdxDstBuf + 1] = position.y / position.w;
    dstVector.data[baseIdxDstBuf + 2] = position.z / position.w;

    baseIdxSrcBuf += 3;
    baseIdxDstBuf += 3;
    vec4 normal = vec4(srcVector.data[baseIdxSrcBuf], srcVector.data[baseIdxSrcBuf + 1], srcVector.data[baseIdxSrcBuf + 2], 0);
    normal =
    weights.x * bonesMatrices.data[drawParameters.bonesMatricesOffset + bonesIndices.x] * normal +
    weights.y * bonesMatrices.data[drawParameters.bonesMatricesOffset + bonesIndices.y] * normal +
    weights.z * bonesMatrices.data[drawParameters.bonesMatricesOffset + bonesIndices.z] * normal +
    weights.w * bonesMatrices.data[drawParameters.bonesMatricesOffset + bonesIndices.w] * normal;
    dstVector.data[baseIdxDstBuf] = normal.x;
    dstVector.data[baseIdxDstBuf + 1] = normal.y;
    dstVector.data[baseIdxDstBuf + 2] = normal.z;

    baseIdxSrcBuf += 3;
    baseIdxDstBuf += 3;
    vec4 tangent = vec4(srcVector.data[baseIdxSrcBuf], srcVector.data[baseIdxSrcBuf + 1], srcVector.data[baseIdxSrcBuf + 2], 0);
    tangent =
    weights.x * bonesMatrices.data[drawParameters.bonesMatricesOffset + bonesIndices.x] * tangent +
    weights.y * bonesMatrices.data[drawParameters.bonesMatricesOffset + bonesIndices.y] * tangent +
    weights.z * bonesMatrices.data[drawParameters.bonesMatricesOffset + bonesIndices.z] * tangent +
    weights.w * bonesMatrices.data[drawParameters.bonesMatricesOffset + bonesIndices.w] * tangent;
    dstVector.data[baseIdxDstBuf] = tangent.x;
    dstVector.data[baseIdxDstBuf + 1] = tangent.y;
    dstVector.data[baseIdxDstBuf + 2] = tangent.z;

    baseIdxSrcBuf += 3;
    baseIdxDstBuf += 3;
    vec4 bitangent = vec4(srcVector.data[baseIdxSrcBuf], srcVector.data[baseIdxSrcBuf + 1], srcVector.data[baseIdxSrcBuf + 2], 0);
    bitangent =
    weights.x * bonesMatrices.data[drawParameters.bonesMatricesOffset + bonesIndices.x] * bitangent +
    weights.y * bonesMatrices.data[drawParameters.bonesMatricesOffset + bonesIndices.y] * bitangent +
    weights.z * bonesMatrices.data[drawParameters.bonesMatricesOffset + bonesIndices.z] * bitangent +
    weights.w * bonesMatrices.data[drawParameters.bonesMatricesOffset + bonesIndices.w] * bitangent;
    dstVector.data[baseIdxDstBuf] = bitangent.x;
    dstVector.data[baseIdxDstBuf + 1] = bitangent.y;
    dstVector.data[baseIdxDstBuf + 2] = bitangent.z;

    baseIdxSrcBuf += 3;
    baseIdxDstBuf += 3;
    vec2 textCoords = vec2(srcVector.data[baseIdxSrcBuf], srcVector.data[baseIdxSrcBuf + 1]);
    dstVector.data[baseIdxDstBuf] = textCoords.x;
    dstVector.data[baseIdxDstBuf + 1] = textCoords.y;
}
```

As you can see the code is very similar to the one used in previous chapters for animation (unrolling the loops). You wil notice that we need to apply an offset for each mesh, since the data is now stored in a common buffer. In order to support push constants in the compute shader. The input / output data is defined as a set of buffers:

* `srcVector`: this buffer will contain vertices information (positions, normals, etc.).
* `weightsVector`: this buffer will contain the weights for the current animation state for a specific mesh and entity.
* `bonesMatrices`: the same but with bones matrices information.
* `dstVector`: this buffer will hold the result of applying animation transformations.

The interesting thing is how we compute that offset. The `gl_GlobalInvocationID` variable will contain the index of work item currently being execute din the compute shader. In our case, we will create as many work items as "chunks" we will have in the global buffer. A chunk models its vertex data, position, normals, texture coordinates etc. Therefore, for vertices data each time the work item is increased, we need to move forward in the buffer 14 positions (14 floats: 3 for positions,. 3 for normals, 3 for bitangents, 3 for tangent and 2 for texture coordinates). The same applies for weights buffers which holds data for weights (4 floats) and bone indices (4 floats) associated to each vertex. We use also the vertex offset to move long the binding poses buffer and the destination buffer along with the `drawParameters` data which point to th ebase offset for each mesh and entity.

We will use this shader in a new class named `AnimationRender` which is defined like this:

```java
package org.lwjglb.engine.graph;

import org.lwjglb.engine.scene.*;

import java.util.*;

import static org.lwjgl.opengl.GL43.*;

public class AnimationRender {

    private ShaderProgram shaderProgram;
    private UniformsMap uniformsMap;

    public AnimationRender() {
        List<ShaderProgram.ShaderModuleData> shaderModuleDataList = new ArrayList<>();
        shaderModuleDataList.add(new ShaderProgram.ShaderModuleData("resources/shaders/anim.comp", GL_COMPUTE_SHADER));
        shaderProgram = new ShaderProgram(shaderModuleDataList);
        createUniforms();
    }

    public void cleanup() {
        shaderProgram.cleanup();
    }

    private void createUniforms() {
        uniformsMap = new UniformsMap(shaderProgram.getProgramId());
        uniformsMap.createUniform("drawParameters.srcOffset");
        uniformsMap.createUniform("drawParameters.srcSize");
        uniformsMap.createUniform("drawParameters.weightsOffset");
        uniformsMap.createUniform("drawParameters.bonesMatricesOffset");
        uniformsMap.createUniform("drawParameters.dstOffset");
    }

    public void render(Scene scene, RenderBuffers globalBuffer) {
        shaderProgram.bind();
        glBindBufferBase(GL_SHADER_STORAGE_BUFFER, 0, globalBuffer.getBindingPosesBuffer());
        glBindBufferBase(GL_SHADER_STORAGE_BUFFER, 1, globalBuffer.getBonesIndicesWeightsBuffer());
        glBindBufferBase(GL_SHADER_STORAGE_BUFFER, 2, globalBuffer.getBonesMatricesBuffer());
        glBindBufferBase(GL_SHADER_STORAGE_BUFFER, 3, globalBuffer.getDestAnimationBuffer());

        int dstOffset = 0;
        for (Model model : scene.getModelMap().values()) {
            if (model.isAnimated()) {
                for (RenderBuffers.MeshDrawData meshDrawData : model.getMeshDrawDataList()) {
                    RenderBuffers.AnimMeshDrawData animMeshDrawData = meshDrawData.animMeshDrawData();
                    Entity entity = animMeshDrawData.entity();
                    Model.AnimatedFrame frame = entity.getAnimationData().getCurrentFrame();
                    int groupSize = (int) Math.ceil((float) meshDrawData.sizeInBytes() / (14 * 4));
                    uniformsMap.setUniform("drawParameters.srcOffset", animMeshDrawData.bindingPoseOffset());
                    uniformsMap.setUniform("drawParameters.srcSize", meshDrawData.sizeInBytes() / 4);
                    uniformsMap.setUniform("drawParameters.weightsOffset", animMeshDrawData.weightsOffset());
                    uniformsMap.setUniform("drawParameters.bonesMatricesOffset", frame.getOffset());
                    uniformsMap.setUniform("drawParameters.dstOffset", dstOffset);
                    glDispatchCompute(groupSize, 1, 1);
                    dstOffset += meshDrawData.sizeInBytes() / 4;
                }
            }
        }

        glMemoryBarrier(GL_SHADER_STORAGE_BARRIER_BIT);
        shaderProgram.unbind();
    }
}
```

As you can see, the definition is quite simple, when creating the shader we need to set up the `GL_COMPUTE_SHADER` to indicate that this is the compute shader. The uniforms that we use will contain the offset in binding pose buffer, weights and matrices buffer and destination buffer. In the `render` method we just iterate over the models and get the mesh draw data for each entity to dispatch a call to the compute shader by invoking the `glDispatchCompute`. The key is to use the `groupSize` variable again. As you can see we need to invoke the shader as many times as vertices chunks there are in the mesh.

## Other changes

We need to update the `SceneRender` class to render the entities associated to animated models. The changes are shown below:

```java
public class SceneRender {
    ...
    private int animDrawCount;
    private int animRenderBufferHandle;
    ...
    public void cleanup() {
        ...
        glDeleteBuffers(animRenderBufferHandle);
    }
    ...
    public void render(Scene scene, RenderBuffers renderBuffers, GBuffer gBuffer) {
        ...
        // Animated meshes
        drawElement = 0;
        for (Model model: scene.getModelMap().values()) {
            if (!model.isAnimated()) {
                continue;
            }
            for (RenderBuffers.MeshDrawData meshDrawData : model.getMeshDrawDataList()) {
                RenderBuffers.AnimMeshDrawData animMeshDrawData = meshDrawData.animMeshDrawData();
                Entity entity = animMeshDrawData.entity();
                String name = "drawElements[" + drawElement + "]";
                uniformsMap.setUniform(name + ".modelMatrixIdx", entitiesIdxMap.get(entity.getId()));
                uniformsMap.setUniform(name + ".materialIdx", meshDrawData.materialIdx());
                drawElement++;
            }
        }
        glBindBuffer(GL_DRAW_INDIRECT_BUFFER, animRenderBufferHandle);
        glBindVertexArray(renderBuffers.getAnimVaoId());
        glMultiDrawElementsIndirect(GL_TRIANGLES, GL_UNSIGNED_INT, 0, animDrawCount, 0);

        glBindVertexArray(0);
        glEnable(GL_BLEND);
        shaderProgram.unbind();
    }

    private void setupAnimCommandBuffer(Scene scene) {
        List<Model> modelList = scene.getModelMap().values().stream().filter(m -> m.isAnimated()).toList();
        int numMeshes = 0;
        for (Model model : modelList) {
            numMeshes += model.getMeshDrawDataList().size();
        }

        int firstIndex = 0;
        int baseInstance = 0;
        ByteBuffer commandBuffer = MemoryUtil.memAlloc(numMeshes * COMMAND_SIZE);
        for (Model model : modelList) {
            for (RenderBuffers.MeshDrawData meshDrawData : model.getMeshDrawDataList()) {
                // count
                commandBuffer.putInt(meshDrawData.vertices());
                // instanceCount
                commandBuffer.putInt(1);
                commandBuffer.putInt(firstIndex);
                // baseVertex
                commandBuffer.putInt(meshDrawData.offset());
                commandBuffer.putInt(baseInstance);

                firstIndex += meshDrawData.vertices();
                baseInstance++;
            }
        }
        commandBuffer.flip();

        animDrawCount = commandBuffer.remaining() / COMMAND_SIZE;

        animRenderBufferHandle = glGenBuffers();
        glBindBuffer(GL_DRAW_INDIRECT_BUFFER, animRenderBufferHandle);
        glBufferData(GL_DRAW_INDIRECT_BUFFER, commandBuffer, GL_DYNAMIC_DRAW);

        MemoryUtil.memFree(commandBuffer);
    }

    public void setupData(Scene scene) {
        ...
        setupAnimCommandBuffer(scene);
        ...
    }
    ...
}
```

The code to render animated models is quite similar as the one used for static entities. The differences is that we are not grouping entities that share the same model, we need to record draw instructions for each of the entities and associated meshes.

We need also to update the `ShadowRender` class to render animated models:

```java
public class ShadowRender {
    ...
    private int animDrawCount;
    private int animRenderBufferHandle;
    ...
    public void cleanup() {
        ...
        glDeleteBuffers(animRenderBufferHandle);
    }
    ...
    public void render(Scene scene, RenderBuffers renderBuffers) {
        ...
        // Animated meshes
        drawElement = 0;
        for (Model model: scene.getModelMap().values()) {
            if (!model.isAnimated()) {
                continue;
            }
            for (RenderBuffers.MeshDrawData meshDrawData : model.getMeshDrawDataList()) {
                RenderBuffers.AnimMeshDrawData animMeshDrawData = meshDrawData.animMeshDrawData();
                Entity entity = animMeshDrawData.entity();
                String name = "drawElements[" + drawElement + "]";
                uniformsMap.setUniform(name + ".modelMatrixIdx", entitiesIdxMap.get(entity.getId()));
                drawElement++;
            }
        }
        glBindBuffer(GL_DRAW_INDIRECT_BUFFER, animRenderBufferHandle);
        glBindVertexArray(renderBuffers.getAnimVaoId());
        for (int i = 0; i < CascadeShadow.SHADOW_MAP_CASCADE_COUNT; i++) {
            glFramebufferTexture2D(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT, GL_TEXTURE_2D, shadowBuffer.getDepthMapTexture().getIds()[i], 0);

            CascadeShadow shadowCascade = cascadeShadows.get(i);
            uniformsMap.setUniform("projViewMatrix", shadowCascade.getProjViewMatrix());

            glMultiDrawElementsIndirect(GL_TRIANGLES, GL_UNSIGNED_INT, 0, animDrawCount, 0);
        }

        glBindVertexArray(0);
    }

    private void setupAnimCommandBuffer(Scene scene) {
        List<Model> modelList = scene.getModelMap().values().stream().filter(m -> m.isAnimated()).toList();
        int numMeshes = 0;
        for (Model model : modelList) {
            numMeshes += model.getMeshDrawDataList().size();
        }

        int firstIndex = 0;
        int baseInstance = 0;
        ByteBuffer commandBuffer = MemoryUtil.memAlloc(numMeshes * COMMAND_SIZE);
        for (Model model : modelList) {
            for (RenderBuffers.MeshDrawData meshDrawData : model.getMeshDrawDataList()) {
                RenderBuffers.AnimMeshDrawData animMeshDrawData = meshDrawData.animMeshDrawData();
                Entity entity = animMeshDrawData.entity();
                // count
                commandBuffer.putInt(meshDrawData.vertices());
                // instanceCount
                commandBuffer.putInt(1);
                commandBuffer.putInt(firstIndex);
                // baseVertex
                commandBuffer.putInt(meshDrawData.offset());
                commandBuffer.putInt(baseInstance);

                firstIndex += meshDrawData.vertices();
                baseInstance++;
            }
        }
        commandBuffer.flip();

        animDrawCount = commandBuffer.remaining() / COMMAND_SIZE;

        animRenderBufferHandle = glGenBuffers();
        glBindBuffer(GL_DRAW_INDIRECT_BUFFER, animRenderBufferHandle);
        glBufferData(GL_DRAW_INDIRECT_BUFFER, commandBuffer, GL_DYNAMIC_DRAW);

        MemoryUtil.memFree(commandBuffer);
    }
}
```

In the `Render` class we just need to instantiate the `AnimationRender` class, and use it in the `render` loop and the `cleanup` method. In the `render` loop we will invoke the `AnimationRender` class `render` method at the very beginning, so animation transformations are applied prior to render the scene.

```java
public class Render {

    private AnimationRender animationRender;
    ...
    public Render(Window window) {
        ...
        animationRender = new AnimationRender();
        ...
    }

    public void cleanup() {
        ...
        animationRender.cleanup();
        ...
    }

    public void render(Window window, Scene scene) {
        animationRender.render(scene, renderBuffers);
        ...
    }    
    ...
}
```

Finally, in the `Main` class we will create two animated entities which will have a different animation update rate to check that we correctly separate per entity information:

```java
public class Main implements IAppLogic {
    ...
    private AnimationData animationData1;
    private AnimationData animationData2;
    ...
    public static void main(String[] args) {
        ...
        Engine gameEng = new Engine("chapter-21", opts, main);
        ...
    }
    ...
    public void init(Window window, Scene scene, Render render) {
        ...
        String bobModelId = "bobModel";
        Model bobModel = ModelLoader.loadModel(bobModelId, "resources/models/bob/boblamp.md5mesh",
                scene.getTextureCache(), scene.getMaterialCache(), true);
        scene.addModel(bobModel);
        Entity bobEntity = new Entity("bobEntity-1", bobModelId);
        bobEntity.setScale(0.05f);
        bobEntity.updateModelMatrix();
        animationData1 = new AnimationData(bobModel.getAnimationList().get(0));
        bobEntity.setAnimationData(animationData1);
        scene.addEntity(bobEntity);

        Entity bobEntity2 = new Entity("bobEntity-2", bobModelId);
        bobEntity2.setPosition(2, 0, 0);
        bobEntity2.setScale(0.025f);
        bobEntity2.updateModelMatrix();
        animationData2 = new AnimationData(bobModel.getAnimationList().get(0));
        bobEntity2.setAnimationData(animationData2);
        scene.addEntity(bobEntity2);
        ...
    }
    ...
    public void update(Window window, Scene scene, long diffTimeMillis) {
        animationData1.nextFrame();
        if (diffTimeMillis % 2 == 0) {
            animationData2.nextFrame();
        }
        ...
    }
}
```

With all of that changes implemented you should be able to see something similar to this.

![Screen shot](<../.gitbook/assets/screenshot (2).png>)
