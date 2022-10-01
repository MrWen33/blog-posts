---
title: dx12笔记(一)-基本概念
date: 2021-09-05 12:34:47
categories: directx12
tags:
- dx12
thumbnail:
---

## 基本概念

* 资源与Resource Heap
* 描述符堆(Descriptor Heap)
* PSO
* Root Signature, Shader
* CommandQueue, CommandList
* SwapChain
* Resource Barrier

## 资源与ResourceHeap
GPU中的资源有Vertex Buffer, Index Buffer, Constant Buffer, 纹理, Render Target等等

GPU中的资源在dx12上以`ID3D12Resource`的形式表示, 而`ID3D12Resource`表示Resource Heap上的一段数据

Resource Heap有四种: 
1. Default Heap (D3D12_HEAP_TYPE_DEFAULT)
2. Upload Heap (D3D12_HEAP_TYPE_UPLOAD)
3. Readback Heap(D3D12_HEAP_TYPE_READBACK)
4. Custom Heap (D3D12_HEAP_TYPE_CUSTOM)

### Default Heap
Default Heap 是一块仅存在于GPU内存上的数据, 所以GPU对其有读写权限, 并且访问非常快速, 而CPU对其没有权限.

### Upload Heap
Upload Heap 在CPU与GPU内存上都存在, CPU有写权限而GPU有读权限. 
Upload Heap一般用于向Default Heap中上传资源

### Readback Heap
Readback Heap 在CPU与GPU内存上都存在, GPU有写权限而CPU有读权限.
用于将渲染的图片或数据从GPU回读到CPU中

## 描述符堆(Descriptor Heap)


## Pipeline State Object

## Root Signature, Shader

## 

## 参考资料

1. [Braynzarsoft: Index Buffer](https://www.braynzarsoft.net/viewtutorial/q16390-directx-12-index-buffers)