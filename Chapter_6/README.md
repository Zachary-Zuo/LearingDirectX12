将src/Shader文件夹移入build目录下



# 一、效果



# 二、笔记

## 2.1、

## 常量缓冲区描述符

常量缓冲区也需要描述符来指定缓冲区中资源的属性。这个描述符是需要创建描述符堆来存放的（和RTV,DSV类似）。

```cpp
UINT objConstSize = CalcConstantBufferByteSize(sizeof(ObjectConstants));
//创建CBV堆
ComPtr<ID3D12DescriptorHeap> cbvHeap;
D3D12_DESCRIPTOR_HEAP_DESC cbHeapDesc;
cbHeapDesc.Type = D3D12_DESCRIPTOR_HEAP_TYPE_CBV_SRV_UAV;
cbHeapDesc.Flags = D3D12_DESCRIPTOR_HEAP_FLAG_SHADER_VISIBLE;
cbHeapDesc.NumDescriptors = 1;
cbHeapDesc.NodeMask = 0;
ThrowIfFailed(d3dDevice->CreateDescriptorHeap(&cbHeapDesc, IID_PPV_ARGS(&cbvHeap)));
```

创建了堆之后，我们就要创建描述符了，而创建描述符的前提是得到常量缓冲区中元素的地址（即不同的单个物体在常量缓冲区上的地址）。所以我们先拿到常量缓冲区的首地址，再通过元素下标计算元素地址。注意：这里不要把描述符堆和常量缓冲区所在的上传堆地址搞混，我们要拿的是uploadBufferResource类中所定义的uploadBuffer地址，所以用unique_ptr这个模板类指针来获取这个类中的数据。

```cpp
//定义并获得物体的常量缓冲区，然后得到其首地址
std::unique_ptr<UploadBufferResource<ObjectConstants>> objCB = nullptr;
//elementCount为1（1个子物体常量缓冲元素），isConstantBuffer为ture（是常量缓冲区）
objCB = std::make_unique<UploadBufferResource<ObjectConstants>>(d3dDevice.Get(), 1, true);
//获得常量缓冲区首地址
D3D12_GPU_VIRTUAL_ADDRESS address;
address = objCB->Resource()->GetGPUVirtualAddress();
//通过常量缓冲区元素偏移值计算最终的元素地址
int cbElementIndex = 0;	//常量缓冲区元素下标
address += cbElementIndex * objConstSize;
//创建CBV描述符
D3D12_CONSTANT_BUFFER_VIEW_DESC cbvDesc;
cbvDesc.BufferLocation = address;
cbvDesc.SizeInBytes = objConstSize;
d3dDevice->CreateConstantBufferView(&cbvDesc, cbvHeap->GetCPUDescriptorHandleForHeapStart());
```

这样，我们就为所需要的数据从CPU传入到GPU做好了准备。之后还要创建根签名，继续将数据绑定到寄存器上，以供着色器访问，进而绘制出几何体。本篇先到这里，未完待续。



## 根签名

我们处理了顶点缓冲区和常量缓冲区，接下来我们要创建根签名。什么是根签名呢？如果把着色器程序看成是一个大函数，那顶点数据和常量数据就是从CPU传入着色器函数的参数，而根签名就好比这些参数的函数签名。所以根签名其实就是将着色器需要用到的数据绑定到对应的寄存器槽上，供着色器访问。根签名分为描述符表、根描述符、根常量，此案例我们使用描述符表来初始化根签名。

```cpp
void D3D12InitApp::BuildRootSignature()
{
	//根参数可以是描述符表、根描述符、根常量
	CD3DX12_ROOT_PARAMETER slotRootParameter[1];
	
   //创建由单个CBV所组成的描述符表
	CD3DX12_DESCRIPTOR_RANGE cbvTable;
	
    cbvTable.Init(D3D12_DESCRIPTOR_RANGE_TYPE_CBV, //描述符类型
		1, //描述符数量
		0);//描述符所绑定的寄存器槽号
	slotRootParameter[0].InitAsDescriptorTable(1, &cbvTable);
	
   //根签名由一组根参数构成
	CD3DX12_ROOT_SIGNATURE_DESC rootSig(1, //根参数的数量
		slotRootParameter, //根参数指针
		0, nullptr, D3D12_ROOT_SIGNATURE_FLAG_ALLOW_INPUT_ASSEMBLER_INPUT_LAYOUT);
	
   //用单个寄存器槽来创建一个根签名，该槽位指向一个仅含有单个常量缓冲区的描述符区域
	ComPtr<ID3DBlob> serializedRootSig = nullptr;
	ComPtr<ID3DBlob> errorBlob = nullptr;
	HRESULT hr = D3D12SerializeRootSignature(&rootSig, D3D_ROOT_SIGNATURE_VERSION_1, &serializedRootSig, &errorBlob);

	if (errorBlob != nullptr)
	{
		OutputDebugStringA((char*)errorBlob->GetBufferPointer());
	}
	ThrowIfFailed(hr);

	ThrowIfFailed(d3dDevice->CreateRootSignature(0,
		serializedRootSig->GetBufferPointer(),
		serializedRootSig->GetBufferSize(),
		IID_PPV_ARGS(&rootSignature)));
}
```



## 输入布局描述和编译着色器字节码

然后我们处理顶点数据的输入布局描述。之前定义了顶点结构体Vertex，我们还需要向DX提供该顶点结构体的描述，使它了解应该怎样处理结构体中的每个成员。这种描述即顶点的输入布局描述。我们先填充D3D12_INPUT_ELEMENT_DESC结构体，可以看到，这里对Vertex结构体中的Pos和Color做了具体的描述。

```cpp
std::vector<D3D12_INPUT_ELEMENT_DESC> inputLayoutDesc;
inputLayoutDesc =
{
      { "POSITION", 0, DXGI_FORMAT_R32G32B32_FLOAT, 0, 0, D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA, 0 },
      { "COLOR", 0, DXGI_FORMAT_R32G32B32A32_FLOAT, 0, 12, D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA, 0 }
};
```

接下来由于我们要编译着色器字节码，所以先把着色器代码写好。DX的着色器语言是HLSL，类似于CG，学过UnityShaderLab的朋友能很快上手。这次最终只是一个Cube的顶点色效果，所以代码中仅仅包含顶点的空间变换和顶点色的输入输出。注意，此处的cbuffer cbPerObject : register(b0)意为：从寄存器槽号为0的地址去访问常量缓冲区中的常量数据。而顶点结构体VertexIn里，数据冒号后面的POSITION和COLOR即为顶点输入布局描述中的定义，由此对应数据才能传入着色器的顶点函数和片元函数。

```cpp
cbuffer cbPerObject : register(b0)
{
	float4x4 gWorldViewProj; 
};

struct VertexIn
{
	float3 PosL  : POSITION;
    float4 Color : COLOR;
};

struct VertexOut
{
	float4 PosH  : SV_POSITION;
    float4 Color : COLOR;
};

VertexOut VS(VertexIn vin)
{
	VertexOut vout;
	
	vout.PosH = mul(float4(vin.PosL, 1.0f), gWorldViewProj);
	
	// Just pass vertex color into the pixel shader.
    vout.Color = vin.Color;
    
    return vout;
}

float4 PS(VertexOut pin) : SV_Target
{
    return pin.Color;
}
```

在DX中，着色器程序必须先被编译成一种可移植的字节码，这样图形驱动程序才能将其继续编译成针对当前系统GPU所优化的本地指令。所以写好了着色器代码后，我们编译它的字节码这里选择运行编译。下面是运行编译函数，我们把它封装后放入TollFunc类。

```cpp
ComPtr<ID3DBlob>  ToolFunc::CompileShader(
	const std::wstring& fileName, 
	const D3D_SHADER_MACRO* defines, 
	const std::string& enteryPoint, 
	const std::string& target)
{
	//若处于调试模式，则使用调试标志
	UINT compileFlags = 0;
#if defined(DEBUG) || defined(_DEBUG)
	//用调试模式来编译着色器 | 指示编译器跳过优化阶段
	compileFlags = D3DCOMPILE_DEBUG | D3DCOMPILE_SKIP_OPTIMIZATION;
#endif // defined(DEBUG) || defined(_DEBUG)

	HRESULT hr = S_OK;

	ComPtr<ID3DBlob> byteCode = nullptr;
	ComPtr<ID3DBlob> errors;
	hr = D3DCompileFromFile(fileName.c_str(), //hlsl源文件名
		defines,	//高级选项，指定为空指针
		D3D_COMPILE_STANDARD_FILE_INCLUDE,	//高级选项，可以指定为空指针
		enteryPoint.c_str(),	//着色器的入口点函数名
		target.c_str(),		//指定所用着色器类型和版本的字符串
		compileFlags,	//指示对着色器断代码应当如何编译的标志
		0,	//高级选项
		&byteCode,	//编译好的字节码
		&errors);	//错误信息

	if (errors != nullptr)
	{
		OutputDebugStringA((char*)errors->GetBufferPointer());
	}
	ThrowIfFailed(hr);

	return byteCode;
}
```

有了编译函数，我们就可以在主文件中编译字节码了。

```cpp
ComPtr<ID3DBlob> vsBytecode = nullptr;
ComPtr<ID3DBlob> psBytecode = nullptr;
HRESULT hr = S_OK;
vsBytecode = ToolFunc::CompileShader(L"Shaders\\color.hlsl", nullptr, "VS", "vs_5_0");
psBytecode = ToolFunc::CompileShader(L"Shaders\\color.hlsl", nullptr, "PS", "ps_5_0");
```

然后我们将输入布局和编译字节码封装成一个函数，命名为BuildByteCodeAndInputLayout函数。





## 构建PSO

再接下来我们要构建PSO（PipeLineStateObject），将之前定义的顶点布局描述、着色器程序字节码、光栅器状态、根签名、图元拓扑方式、采样方式、混合方式、深度模板状态、RTV格式、DSV格式等等对象绑定到图形流水线上。

```cpp
void D3D12InitApp::BuildPSO()
{
	D3D12_GRAPHICS_PIPELINE_STATE_DESC psoDesc = {};
	ZeroMemory(&psoDesc, sizeof(D3D12_GRAPHICS_PIPELINE_STATE_DESC));
	psoDesc.InputLayout = { inputLayoutDesc.data(), (UINT)inputLayoutDesc.size() };
	psoDesc.pRootSignature = rootSignature.Get();
	psoDesc.VS = { reinterpret_cast<BYTE*>(vsBytecode->GetBufferPointer()), vsBytecode->GetBufferSize() };
	psoDesc.PS = { reinterpret_cast<BYTE*>(psBytecode->GetBufferPointer()), psBytecode->GetBufferSize() };
	psoDesc.RasterizerState = CD3DX12_RASTERIZER_DESC(D3D12_DEFAULT);
	psoDesc.BlendState = CD3DX12_BLEND_DESC(D3D12_DEFAULT);
	psoDesc.DepthStencilState = CD3DX12_DEPTH_STENCIL_DESC(D3D12_DEFAULT);
	psoDesc.SampleMask = UINT_MAX;	//0xffffffff,全部采样，没有遮罩
	psoDesc.PrimitiveTopologyType = D3D12_PRIMITIVE_TOPOLOGY_TYPE_TRIANGLE;
	psoDesc.NumRenderTargets = 1;
	psoDesc.RTVFormats[0] = DXGI_FORMAT_R8G8B8A8_UNORM;	//归一化的无符号整型
	psoDesc.DSVFormat = DXGI_FORMAT_D24_UNORM_S8_UINT;
	psoDesc.SampleDesc.Count = 1;	//不使用4XMSAA
	psoDesc.SampleDesc.Quality = 0;	////不使用4XMSAA

	ThrowIfFailed(d3dDevice->CreateGraphicsPipelineState(&psoDesc, IID_PPV_ARGS(&PSO)));
}
```





## Draw

rawIndexedInstanced函数，即通过索引缓冲区来绘制几何体，因为此案例中没有子物体，所以第4个参数为0，如果有子物体这里将做偏移计算。

```cpp
//设置CBV描述符堆
ID3D12DescriptorHeap* descriHeaps[] = { cbvHeap.Get() };//注意这里之所以是数组，是因为还可能包含SRV和UAV，而这里我们只用到了CBV
cmdList->SetDescriptorHeaps(_countof(descriHeaps), descriHeaps);
//设置根签名
cmdList->SetGraphicsRootSignature(rootSignature.Get());
//设置顶点缓冲区
cmdList->IASetVertexBuffers(0, 1, &GetVbv());
//将图元拓扑类型传入流水线
cmdList->IASetPrimitiveTopology(D3D_PRIMITIVE_TOPOLOGY_TRIANGLELIST);
//设置根描述符表
cmdList->SetGraphicsRootDescriptorTable(0, //根参数的起始索引
	cbvHeap->GetGPUDescriptorHandleForHeapStart());

//绘制顶点（通过索引缓冲区绘制）
cmdList->DrawIndexedInstanced(sizeof(indices), //每个实例要绘制的索引数
	1,	//实例化个数
	0,	//起始索引位置
	0,	//子物体起始索引在全局索引中的位置
	0);	//实例化的高级技术，暂时设置为0
```





## 2.2、控制相机

```cpp
void D3D12InitApp::OnMouseDown(WPARAM btnState, int x, int y)
{
	lastMousePos.x = x;	//按下的时候记录坐标x分量
	lastMousePos.y = y;	//按下的时候记录坐标y分量

	SetCapture(mhMainWnd);	//在属于当前线程的指定窗口里，设置鼠标捕获
}

void D3D12InitApp::OnMouseUp(WPARAM btnState, int x, int y)
{
	ReleaseCapture();	//按键抬起后释放鼠标捕获
}

void D3D12InitApp::OnMouseMove(WPARAM btnState, int x, int y)
{
	if ((btnState & MK_LBUTTON) != 0)//如果在左键按下状态
	{
		//将鼠标的移动距离换算成弧度，0.25为调节阈值
		float dx = XMConvertToRadians(static_cast<float>(lastMousePos.x-x) * 0.25f);
		float dy = XMConvertToRadians(static_cast<float>(lastMousePos.y-y) * 0.25f);
		//计算鼠标没有松开前的累计弧度
		theta += dx;
		phi += dy;
		//限制角度phi的范围在（0.1， Pi-0.1）
		theta = MathHelper::Clamp(theta, 0.1f, 3.1416f - 0.1f);
	}
	else if ((btnState & MK_RBUTTON) != 0)//如果在右键按下状态
	{
		//将鼠标的移动距离换算成缩放大小，0.005为调节阈值
		float dx = 0.005f * static_cast<float>(x - lastMousePos.x);
		float dy = 0.005f * static_cast<float>(y - lastMousePos.y);
		//根据鼠标输入更新摄像机可视范围半径
		radius += dx - dy;
		//限制可视范围半径
		radius = MathHelper::Clamp(radius, 1.0f, 20.0f);
	}
	//将当前鼠标坐标赋值给“上一次鼠标坐标”，为下一次鼠标操作提供先前值
	lastMousePos.x = x;
	lastMousePos.y = y;
}
```



```cpp
float theta = 1.5*XM_PI;
float phi = XM_PIDIV4;
float radius = 10.0f;

float y = radius * cosf(phi);
float x = radius * sinf(phi) * cosf(theta);
float z = radius * sinf(phi) * sinf(theta);
```





# 三、练习

## 3.1 第一题

```cpp
std::vector<D3D12_INPUT_ELEMENT_DESC> inputLayoutDesc;
inputLayoutDesc =
{
    { "POSITION", 0, DXGI_FORMAT_R32G32B32_FLOAT,    0, 0,  D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA, 0 },
    { "TANGENT",  0, DXGI_FORMAT_R32G32B32_FLOAT,    0, 12, D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA, 0 },
    { "NORMAL",   0, DXGI_FORMAT_R32G32B32_FLOAT,    0, 24, D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA, 0 },
    { "TEXCOORD", 0, DXGI_FORMAT_R32G32_FLOAT,       0, 36, D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA, 0 },
    { "TEXCOORD", 1, DXGI_FORMAT_R32G32_FLOAT,       0, 44, D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA, 0 },
    { "COLOR",    0, DXGI_FORMAT_R32G32B32A32_FLOAT, 0, 52, D3D12_INPUT_CLASSIFICATION_PER_VERTEX_DATA, 0 }
};
```

- 先计算bit，再转换为byte，例如：R32G32B32的大小为32*3=96bit，也就是12个byte。
- 第二套纹理坐标的语义索引要+1



## 3.2 第二题

主要修改`IASetVertexBuffers`绑定的资源、布局以及着色器代码，BoxApp.cpp中查询“Change”即可找到修改的代码部分

![Exercise_2](\Image\Exercise_2.png)



## 3.3 第三题

只需修改`mCommandList->IASetPrimitiveTopology()`的参数即可

a、点列表：`D3D_PRIMITIVE_TOPOLOGY_POINTLIST`

![Exercise_3_1](\Image\Exercise_3_1.png)

b、线条带：`D3D_PRIMITIVE_TOPOLOGY_LINELIST`

![Exercise_3_2](\Image\Exercise_3_2.png)

c、线列表：`D3D_PRIMITIVE_TOPOLOGY_LINESTRIP`

![Exercise_3_3](\Image\Exercise_3_3.png)

d、三角形带：`D3D_PRIMITIVE_TOPOLOGY_TRIANGLELIST`

![Exercise_3_4](\Image\Exercise_3_4.png)

e、三角形列表：`D3D_PRIMITIVE_TOPOLOGY_TRIANGLESTRIP`

![Exercise_3_5](Image\Exercise_3_5.png)



## 3.4 第四题

关键在于确定三棱锥5个顶点的坐标，以及三角形列表的索引排序。假设四棱锥的中心为（0， 0， 0），顶点为（0，1，0），即可得到其它所有点的坐标，顶点颜色根据需求初始化即可。

```cpp
std::array<VertexPosition, 5> positionVertices =
{
	VertexPosition({ XMFLOAT3(+0.0f, +1.0f, +0.0f) }),
	VertexPosition({ XMFLOAT3(-1.0f, -1.0f, -1.0f) }),
	VertexPosition({ XMFLOAT3(+1.0f, -1.0f, -1.0f) }),
	VertexPosition({ XMFLOAT3(+1.0f, -1.0f, +1.0f) }),
	VertexPosition({ XMFLOAT3(-1.0f, -1.0f, +1.0f) })
};
std::array<VertexColor, 5> colorVertices =
{
	VertexColor({ XMFLOAT4(Colors::Red) }),
	VertexColor({ XMFLOAT4(Colors::Green) }),
	VertexColor({ XMFLOAT4(Colors::Green) }),
	VertexColor({ XMFLOAT4(Colors::Green) }),
	VertexColor({ XMFLOAT4(Colors::Green) })
};
```

在确定索引顺序时，由于DX是左手坐标系，所以索引绕序为顺时针，这一点要注意。

```cpp
std::array<std::uint16_t, 18> indices =
{
	//前
	0, 2, 1,

	//后
	0, 4, 3,

	//左
	0, 1, 4,

	//右
	0, 3, 2,

	//下
	2, 4, 1,
	2, 3, 4
};
```

![Exercise_4](Image\Exercise_4.png)