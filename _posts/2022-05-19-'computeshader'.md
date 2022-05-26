---
title:  "[Unity] Compute Shader를 활용해 텍스처의 그라디언트 구현하기"
excerpt: 

categories: Unity
tags:
  # - [math]

toc: true
toc_sticky: true

date: 2022-05-19
last_modified_at: 2022-05-19

sidebar_main: true

---

- 먼저, GPU에서 실행될 Compute Shader 파일을 생성한다.

    ```csharp
    #pragma kernel KernelFunctionA
    #pragma kernel KernelFunctionB

    RWTexture2D<float4> result;

    [numthreads(8,8,1)]
    void KernelFunctionA(uint3 dispatchThreadID : SV_DispatchThreadID)
    {
        float width;
        float height;

        result.GetDimensions(width, height);
        result[dispatchThreadID.xy] = float4(dispatchThreadID.x / width, dispatchThreadID.x / width, dispatchThreadID.x / width, 1);  
    }

    [numthreads(8,8,1)]
    void KernelFunctionB(uint3 dispatchThreadID : SV_DispatchThreadID)
    {
        float width;
        float height;

        result.GetDimensions(width, height);
        result[dispatchThreadID.xy] = float4(dispatchThreadID.y / height, dispatchThreadID.y / height, dispatchThreadID.y / height, 1);
    }
    ```

- #pragma 지시어로 어느 함수가 compute shader의 커널로 컴파일될지를 지정한다.
- 텍스처 계산에 쓰일 커널 함수인 KernelFunctionA와 KernelFunctionB를 명시해 준다.

    ```csharp
    #pragma kernel KernelFunctionA
    #pragma kernel KernelFunctionB
    ```

- 읽고 쓸 수 있는(RW) 2D 텍스처 버퍼 result를 선언한다. 이 버퍼는 RGBA값을 저장할 수 있는 float4형으로 구성되어 있다. 이렇게 해야 GPU에서 계산한 결과를 CPU 쪽으로 넘겨줄 수 있고, 스크립트에서 받아서 처리할 수 있다.

    ```csharp
    RWTexture2D<float4> result;
    ```

- 커널함수 구현 시, numthreads attribute로 몇 개의 스레드가 이 커널을 실행할지를 3차원으로 지정해야 한다.
- 함수 파라미터는 필수적으로 Unity Compute Shader에 정의된 compute shader semantics 타입을 따라야 하며, 이름은 자유롭게 정의 가능하다.
- semantics
    - SV_GroupID (uint3)
        - 현재 커널을 실행하는 스레드가 속한 Group의 3차원 인덱스
    - SV_GroupThreadID (uint3)
        - Group 내의 개별 Thread의 3차원 인덱스
    - SV_GroupIndex (uint)
        - Group 내의 개별 Thread의 1차원 인덱스
    - **SV_DispatchThreadID (uint)**
        - SV_GroupID * numthreads + GroupThreadID
        - 현재 커널을 실행하는 스레드의 global 인덱스 (전체 group을 통틀어, 전체 스레드 중 자신이 몇 번째인지를 나타냄)

    ```csharp
    [numthreads(8,8,1)]
    void KernelFunctionA(uint3 dispatchThreadID : SV_DispatchThreadID)
    {
        float width;
        float height;

        result.GetDimensions(width, height);
        result[dispatchThreadID.xy] = float4(dispatchThreadID.x / width, dispatchThreadID.x / width, dispatchThreadID.x / width, 1);  
    }

    ```

- KernelFunctionA는 SV_DispatchThreadID 타입의 파라미터 dispatchThreadID를 받는다.
- Texture2D의 GetDimensions 메서드로 텍스처 버퍼의 너비와 높이를 받아와 width와 height에 저장한다.
- dispatchThreadID의 x와 y값(uint2)을 가져와 텍스처 버퍼의 인덱스로 사용하고, 해당 인덱스의 요소에 float4 타입의 RGBA 값을 넣어준다.
    - 그 값은 dispatchThreadID에 따라 결정되는데, R, G, B값은 dispatchThreadID.x / width로, 
    워크 그룹 수가 (텍스처 가로사이즈 / x차원 스레드 개수, 텍스처 세로사이즈 / y차원 스레드 개수, 1)일 경우 
    dispatchThreadID.x는 0~(텍스처 가로사이즈)의 값을 가지게 된다.
    - 고로 스레드들은 dispatchThreadID.x에 따라 0~1의 값을 만들어내며, 같은 x값을 가진 텍스처 버퍼의 요소들은 같은 값으로 채워진다.
    - 결과적으로 dispatchThreadID.x의 증가에 따라 RGBA 값이 (0, 0, 0, 1)에서 (1, 1, 1, 1)로 변화하며, 가로 방향 그라디언트(오른쪽으로 갈수록 흰색에 가까워짐)가 표현된다.
- 다음은 width와 height가 32이고 워크 그룹 수가 (4, 4, 1)일 때 스레드들에 대응되는 텍스처 버퍼 result의 요소들을 나타낸다.
    
![1](https://user-images.githubusercontent.com/90246317/170466962-ec7a6978-66c5-4482-a932-307d2bad663d.png)
    
- KernelFunctionB의 로직은 KernelFunctionA와 동일하지만, dispatchThread.y / height를 RGB값에 대입함으로서 세로 방향 그라디언트(위쪽으로 갈수록 흰색에 가까워짐)를 만들어낸다.

    ```csharp
    [numthreads(8,8,1)]
    void KernelFunctionB(uint3 dispatchThreadID : SV_DispatchThreadID)
    {
        float width;
        float height;

        result.GetDimensions(width, height);
        result[dispatchThreadID.xy] = float4(dispatchThreadID.y / height, dispatchThreadID.y / height, dispatchThreadID.y / height, 1);
    }
    ```

- 다음으로, C# 스크립트를 작성한다. 이 스크립트는 Compute Shader의 계산 결과를 가져와 텍스처에 적용하고, 그 텍스처를 게임오브젝트의 머티리얼에 적용한다.

    ```csharp
    using UnityEngine;

    struct ThreadSize
    {
        public int X { get; set; }
        public int Y { get; set; }
        public int Z { get; set; }

        public ThreadSize(uint x, uint y, uint z)
        {
            X = (int) x;
            Y = (int) y;
            Z = (int) z;
        }
    }

    public class NewBehaviourScript : MonoBehaviour
    {
        [SerializeField] private GameObject planeA;
        [SerializeField] private GameObject planeB;

        [SerializeField] private ComputeShader computeShader;

        private RenderTexture _renderTextureA;
        private RenderTexture _renderTextureB;

        private int _kernelIndexA;
        private int _kernelIndexB;

        void Start()
        {
            // planeA의 텍스처 - 가로 방향 그라디언트
            _renderTextureA = new RenderTexture(512, 512, 1, RenderTextureFormat.ARGB32);
            _renderTextureA.enableRandomWrite = true;
            _renderTextureA.Create();

            _kernelIndexA = computeShader.FindKernel("KernelFunctionA");
            computeShader.GetKernelThreadGroupSizes(_kernelIndexA, out uint threadSizeX, out uint threadSizeY, out uint threadSizeZ);
            ThreadSize threadSizeA = new ThreadSize(threadSizeX, threadSizeY, threadSizeZ);

            computeShader.SetTexture(_kernelIndexA, "result", _renderTextureA);
            computeShader.Dispatch(_kernelIndexA, _renderTextureA.width / threadSizeA.X, _renderTextureA.height / threadSizeA.Y, _renderTextureA.depth / threadSizeA.Z);

            planeA.GetComponent<Renderer>().material.mainTexture = _renderTextureA;

            // planeB의 텍스처 - 세로 방향 그라디언트
            _renderTextureB = new RenderTexture(512, 512, 1, RenderTextureFormat.ARGB32);
            _renderTextureB.enableRandomWrite = true;
            _renderTextureB.Create();

            _kernelIndexB = computeShader.FindKernel("KernelFunctionB");
            computeShader.GetKernelThreadGroupSizes(_kernelIndexB, out threadSizeX, out threadSizeY, out threadSizeZ);
            ThreadSize threadSizeB = new ThreadSize(threadSizeX, threadSizeY, threadSizeZ);

            computeShader.SetTexture(_kernelIndexB, "result", _renderTextureB);
            computeShader.Dispatch(_kernelIndexB, _renderTextureB.width / threadSizeB.X, _renderTextureB.height / threadSizeB.Y, _renderTextureB.depth / threadSizeB.Z);

            planeB.GetComponent<Renderer>().material.mainTexture = _renderTextureB;
        }
    }
    ```

- 먼저, 하나의 그룹을 구성하는 스레드들의 수를 커널함수별로 저장해두기 위해 구조체를 하나 만든다.
    
    ```csharp
    using UnityEngine;
    
    struct ThreadSize
    {
        public int X { get; set; }
        public int Y { get; set; }
        public int Z { get; set; }
    
        public ThreadSize(uint x, uint y, uint z)
        {
            X = (int) x;
            Y = (int) y;
            Z = (int) z;
        }
    }
    ```
    
- 그리고 텍스처를 적용할 게임오브젝트와  Compute Shader 에셋을 참조하기 위해 다음과 같이 MonoBehaviour를 상속받는 클래스 안에 필드를 선언한다. (인스펙터 상에만 노출되도록 SerializeField 어트리뷰트를 붙여주었다.)
    
    ```csharp
    public class NewBehaviourScript : MonoBehaviour
    {
        [SerializeField] private GameObject planeA;
        [SerializeField] private GameObject planeB;
    
        [SerializeField] private ComputeShader computeShader;
    
    ```
    
- 다음으로 planeA, planeB에 적용할 텍스처를 선언한다. 이 텍스처는 Compute Shader의 텍스처 버퍼 result와 연결되며, 커널함수 KernelFunctionA와 KernelFunctionB의 실행(Dispatch)를 통해 계산된 값을 받아온다.
- Dispatch와 버퍼 설정 등을 위해 커널의 인덱스를 저장해둘 int 필드를 두 개 선언한다.
    
    ```csharp
        private RenderTexture _renderTextureA;
        private RenderTexture _renderTextureB;
    
        private int _kernelIndexA;
        private int _kernelIndexB;
    ```
    
- 커널함수의 실행과 텍스처 적용은 게임 시작 시 한 번만으로 충분하므로 Start 함수에 다음 코드를 작성한다.
    
    ```csharp
     void Start()
        {
            // planeA의 텍스처 - 가로 방향 그라디언트
            _renderTextureA = new RenderTexture(512, 512, 1, RenderTextureFormat.ARGB32);
            _renderTextureA.enableRandomWrite = true;
            _renderTextureA.Create();
    
            _kernelIndexA = computeShader.FindKernel("KernelFunctionA");
            computeShader.GetKernelThreadGroupSizes(_kernelIndexA, out uint threadSizeX, out uint threadSizeY, out uint threadSizeZ);
            ThreadSize threadSizeA = new ThreadSize(threadSizeX, threadSizeY, threadSizeZ);
    
            computeShader.SetTexture(_kernelIndexA, "result", _renderTextureA);
            computeShader.Dispatch(_kernelIndexA, _renderTextureA.width / threadSizeA.X, _renderTextureA.height / threadSizeA.Y, _renderTextureA.depth / threadSizeA.Z);
    
            planeA.GetComponent<Renderer>().material.mainTexture = _renderTextureA;
    
    ```
    
- 먼저 _renderTextureA가 참조할 새로운 텍스처를 만드는데, width와 height를 지정하고(512) 포맷은 채널당 8bit를 사용하는 ARGB32로 설정한다. (2차원 텍스처이므로 depth는 1이다.)
    - enableRandomWrite 프로퍼티에 true를 대입해 셰이더에서 텍스처에 접근하고 값을 수정할 수 있게 설정해준다.
    - Create 메서드로 실질적인 텍스처를 생성한다.
- Compute Shader에서 KernelFunctionA라는 이름의 커널함수를 찾아 _kernelIndexA에 캐싱해둔다.
- 그 다음, 얻은 인덱스를 통해 그 커널함수의 numthreads를 uint 변수 세 개로 받아오고, 아까 만든 Threadsize 구조체의 새 인스턴스 threadSizeA를 생성해 세 변수의 값으로 초기화해 둔다.
- SetTexture 메서드로 KernelFunctionA 함수가 실행될 때 텍스처 버퍼 result에 _renderTextureA가 연결되도록 설정한 다음, Dispatch 메서드를 통해 커널을 실행한다.
- 워크 그룹의 수는 Dispatch 메서드를 실행할 때 결정하는데, 텍스처의 RGB값이 가로 방향으로 0에서 1까지 증가하도록 하기 위해서는 워크 그룹 수를 (텍스처 가로사이즈 / x축 스레드 수, 텍스처 세로사이즈 / y축 스레드 수, 텍스처 깊이 / z축 스레드 수(여기서는 별로 중요하지 않음))으로 설정해줘야 한다. 그래야 dispatchThreadID의 x와 y값이 0에서 512까지로 설정되기 때문이다.
- 커널함수가 실행되었으므로 _renderTextureA에는 계산 결과가 저장되었다. 이제 planeA의 렌더러 컴포넌트를 가져와 머티리얼의 메인 텍스처에 _renderTextureA를 집어넣으면, planeA의 텍스처는 Compute Shader에 의해 계산된 그라디언트로 표시된다.
- 두 번째 게임오브젝트 planeB에 적용할 텍스처를 위해 KernelFunctionB를 실행하는 과정도 A와 아주 유사한 과정을 거친다.
    
    ```csharp
            // planeB의 텍스처 - 세로 방향 그라디언트
            _renderTextureB = new RenderTexture(512, 512, 1, RenderTextureFormat.ARGB32);
            _renderTextureB.enableRandomWrite = true;
            _renderTextureB.Create();
    
            _kernelIndexB = computeShader.FindKernel("KernelFunctionB");
            computeShader.GetKernelThreadGroupSizes(_kernelIndexB, out threadSizeX, out threadSizeY, out threadSizeZ);
            ThreadSize threadSizeB = new ThreadSize(threadSizeX, threadSizeY, threadSizeZ);
    
            computeShader.SetTexture(_kernelIndexB, "result", _renderTextureB);
            computeShader.Dispatch(_kernelIndexB, _renderTextureB.width / threadSizeB.X, _renderTextureB.height / threadSizeB.Y, _renderTextureB.depth / threadSizeB.Z);
    
            planeB.GetComponent<Renderer>().material.mainTexture = _renderTextureB;
        }
    }
    ```
    
- 빈 게임오브젝트를 만들고 만든 스크립트를 컴포넌트로 등록한 후, planeA와 planeB, Compute Shader에 각각 Plane 게임오브젝트들과 셰이더 에셋을 드래그 앤 드랍한다.
    
    ![2](https://user-images.githubusercontent.com/90246317/170466850-8f489f3f-1c70-483b-89b0-c5f0cb93b3ba.png)    
- 플레이 버튼을 눌러 실행하면 텍스처가 표시된다!
    
    ![3](https://user-images.githubusercontent.com/90246317/170466888-892b8e7e-3de4-4d80-9df7-e715c9531372.png)
