---
title: dx12笔记(二)-资源的创建，上传与使用
date: 2021-09-05 14:05:44
category: directx12
tag:
  - dx12
---


## 前置概念
DX12中的资源包括Vertex Buffer, Index Buffer, Constant Buffer, Texture等等

GPU端的DX12资源的C++表示为`ID3DResource`, 表示资源堆中的一段内存.

CPU端的DX12资源C++表示为`ID3DBlob`

Resource State与Resource Barrier: Resource State意思为资源状态，一些对资源的操作在某些状态下可以进行而在某些状态下不行。Resource Barrier用于资源状态的转换，这样可以避免资源的操作与当前状态不符. Resource Barrier进行的转换操作在Command List执行.

## Vertex Buffer, Index Buffer

### 创建
Vertex/Index Buffer很少改变，所以使用Default Heap存储数据，使用Upload Heap将数据上传到Default Heap中

数据流动: 

使用`CreateCommittedResource`创建隐式Default Heap与资源

关于`CreateCommitResource`, [MSDN](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12device-createcommittedresource)上的说明是：创建资源与隐式堆(implicit heap)，这个隐式堆大小可以容纳整个资源, 而创建的资源资源被映射到隐式堆.
```C++
ThrowIfFailed(device->CreateCommittedResource(
    &CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_DEFAULT), // 堆的类型：Default
    D3D12_HEAP_FLAG_NONE,
    &CD3DX12_RESOURCE_DESC::Buffer(byteSize), // 资源的大小
    D3D12_RESOURCE_STATE_COMMON,  // 初始状态
    nullptr,
    IID_PPV_ARGS(defaultBuffer.GetAddressOf()))); // 创建的资源，通过引用参数方式传入. defualtBuffer类型为ComPtr<ID3DResource>
```
同样使用`CreateComittedResource`创建Upload Heap
```C++
ThrowIfFailed(device->CreateCommittedResource(
    &CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_UPLOAD), // 堆类型：Upload
    D3D12_HEAP_FLAG_NONE,
    &CD3DX12_RESOURCE_DESC::Buffer(byteSize),
    D3D12_RESOURCE_STATE_GENERIC_READ,
    nullptr,
    IID_PPV_ARGS(uploadBuffer.GetAddressOf())));
```

创建完毕后，使用Resource Barrier转换资源状态并使用`UpdateSubresources`方法通过Upload Heap将数据上传到Default Heap.

```C++
// 准备数据
D3D12_SUBRESOURCE_DATA subResourceData = {};
subResourceData.pData = initData;
subResourceData.RowPitch = byteSize;
subResourceData.SlicePitch = subResourceData.RowPitch;

// 转换状态上传数据
cmdList->ResourceBarrier(1, &CD3DX12_RESOURCE_BARRIER::Transition(defaultBuffer.Get(), 
    D3D12_RESOURCE_STATE_COMMON, D3D12_RESOURCE_STATE_COPY_DEST));
UpdateSubresources<1>(cmdList, defaultBuffer.Get(), uploadBuffer.Get(), 0, 0, 1, &subResourceData);
cmdList->ResourceBarrier(1, &CD3DX12_RESOURCE_BARRIER::Transition(defaultBuffer.Get(),
    D3D12_RESOURCE_STATE_COPY_DEST, D3D12_RESOURCE_STATE_GENERIC_READ));
```

这样就完成了将数据上传到Default Heap的工作

### 使用
使用`ID3DResource`创建VertexBufferView(`D3D12_VERTEX_BUFFER_VIEW`)/IndexBufferView(`D3D12_INDEX_BUFFER_VIEW`), 将其使用Command List的方法`IASetVertexBuffers`/`IASetIndexBuffer`绑定即可在之后的`DrawIndexedInstanced`中使用对应的VertexBuffer/IndexBuffer

```C++
// 创建VertexBufferView, IndexBufferView
D3D12_VERTEX_BUFFER_VIEW vbv;
vbv.BufferLocation = vertexBuffer->GetGPUVirtualAddress(); // 先前创建的defaultBuffer
vbv.StrideInBytes = VertexByteStride; // Vertex结构体的大小
vbv.SizeInBytes = VertexBufferByteSize; // VertexBuffer总大小

D3D12_INDEX_BUFFER_VIEW ibv;
ibv.BufferLocation = indexBuffer->GetGPUVirtualAddress();
ibv.Format = DXGI_FORMAT_R16_UINT; // Index Buffer数据格式
ibv.SizeInBytes = IndexBufferByteSize; // Index Buffer总大小

// ...

// 设置VBV, IBV
cmdList->IASetVertexBuffers(0, 1, &vbv);
cmdList->IASetIndexBuffer(&ibv);

// ...
// cmdList->DrawIndexedInstanced(...)

```



## Constant Buffer

### 创建
Constant Buffer是更改很频繁的数据, 所以直接使用Upload Heap向GPU提供数据.

把相同类型的constant buffer存在同一个Heap中, 需要注意GPU只能读取地址为256倍数的constant buffer数据, 所以要对constant buffer的大小进行手动对齐

```C++
int elementByteSize = (sizeof(ConstantBuffer) + 0xFF) & ~0xFF // 256字节对齐

// 创建Upload Heap
ThrowIfFailed(device->CreateCommittedResource(
    &CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_UPLOAD),
    D3D12_HEAP_FLAG_NONE,
    &CD3DX12_RESOURCE_DESC::Buffer(elementByteSize*elementCount),
    D3D12_RESOURCE_STATE_GENERIC_READ,
    nullptr,
    IID_PPV_ARGS(&uploadBuffer)));

// Upload Heap映射到CPU端数据
ThrowIfFailed(uploadBuffer->Map(0, nullptr, reinterpret_cast<void**>(&mappedData)));
```

Upload Buffer映射到CPU端数据后, 直到释放都不会Unmap, 所以需要用户自己保证GPU读取数据的同时CPU不在写入(使用同步技术).

若要修改constant buffer数据, 直接修改Upload Buffer映射到的CPU端数据即可

```C++
memcpy(&mappedData[elementIndex*elementByteSize], &newData, sizeof(ConstantBuffer)); // mappedData类型为BYTE*
```

### 使用

C++端:
在Root Signature中设置后, 渲染循环中绑定对应的Root Signature, 使用偏移量设置对应的数据

```C++
// 定义ConstantBuffer结构
struct ConstantBuffer{
    XMFLOAT4X4 world;
    XMFLOAT4x4 view;
    XMFLOAT4x4 proj;
};

// 设置Root Signature

// ...
// slotRootParameter[0].InitAs...
slotRootParameter[1].InitAsConstantBufferView(0); // 可以直接设置CBV参数, 也可以放在Descriptor Heap中. 参数为ShaderRegister编号

// ...
// CD3DX12_ROOT_SIGNATURE_DESC rootSigDesc(..., slotRootParameter, ...)
// ...
// D3D12SerializeRootSignature(&rootSigDesc, ...)
```

```C++
// 渲染循环中
int elementByteSize = (sizeof(ConstantBuffer) + 0xFF) & ~0xFF // 256字节对齐
D3D12_GPU_VIRTUAL_ADDRESS cbAddress = cbUploadBuffer->GetGPUVirtualAddress() + elementIndex * elementByteSize;
cmdList->SetGraphicsRootConstantBufferView(1, cbAddress);

// ...
```

HLSL端:

```HLSL
cbuffer cb: register(b0) // 与RootSignature中设置的ShaderRegister对应
{
    float4x4 g_World;
    float4x4 g_View;
    float4x4 g_Proj;
};
```
或使用dx12的新语法
```HLSL
struct Constants{
    float4x4 World;
    float4x4 View;
    float4x4 Proj;
};

ConstantBuffer<Constants> gConstants: register(b0);
```

## Texture

### 创建
与Vertex Buffer/Index Buffer相似, 经过Upload Heap向Default Heap上传从文件中读取或程序化生成的数据

不同的点是ResourceBarrier的目标不是`D3D12_RESOURCE_STATE_GENERIC_READ`而是`D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE`
### 使用
与CBuffer相似, 在Descriptor Heap上创建ShaderResourceView, 设置RootSignature, 渲染循环中绑定, 在HLSL中使用.

PS. RootSignature不支持使用`SetGraphicsShaderResourceView`直接绑定材质贴图, 所以只能使用Descriptor Heap的方式绑定Texture.

PPS. 支持使用`SetGraphicsShaderResourceView`绑定的是只读缓冲区, 比如`Buffer<float4>`或`StructuredBuffer<T>`.

## 参考资料
1. DX12龙书: *Introduction To 3D Game Programming With DirectX12*
2. [Braynzar Soft DX12 tutorals](https://www.braynzarsoft.net/viewtutorial/q16390-04-directx-12-braynzar-soft-tutorials)
3. [MSDN](https://docs.microsoft.com/en-us/windows/win32/api/d3d12/)
