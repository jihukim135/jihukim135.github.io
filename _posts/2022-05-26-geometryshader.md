---
title:  "[Unity] Geometry Shader 기초"
excerpt: Quad를 직육면체로 바꿔 보자

categories: Unity
tags:

toc: true
toc_sticky: true

date: 2022-05-26
last_modified_at: 2022-05-26

sidebar_main: true

---

## Geometry Shader

![1](https://user-images.githubusercontent.com/90246317/170819193-278b88a9-7514-4203-9fc1-c51c5cb3ae17.png){: .align-center}

- 렌더링 파이프라인 상에서 Vertex Shader와 Tessellator 다음, Rasterizer와 Fragment Shader 이전에 위치한다.
    - 선택적으로 사용되는, 필수가 아닌 셰이더 단계이다.
- GPU를 이용해 동적으로 점이나 선, 삼각형 등의 도형을 생성 및 삭제, 변경할 수 있는 기능을 가진다. (Vertex Shader는 할 수 없다.)
    - 즉, 하나의 정점 입력에 대해 복수의 정점 출력이 가능하다.
    - 출력될 정점의 수는 maxvertexcount 어트리뷰트로 사전에 설정되어야 한다.
- Vertex Shader에서 입력받은 도형의 정보를 처리하는데, 이 도형의 정보(기본 도형)은 사용자가 정의 가능하다.
    - 기본 도형이 삼각형인 경우 정점 3개, 선인 경우 정점 2개, 점인 경우 정점 1개의 정보가 입력된다.
    - 기본 도형 단위로 처리되는데, 예를 들어 메시가 삼각형 2개로 구성되어 있고 기본 도형이 삼각형으로 설정되었을 경우 Geometry Shader는 삼각형마다 한 번씩, 총 2회 실행된다.
- Geometry Shader의 출력은 기본 도형을 구성하는 정점들이다.
    - Rasterizer는 Geometry Shader에서 출력된 값만 처리하므로, 보존되어야 할 Vertex Shader의 출력 정보는 Geometry Shader에서 함께 출력해줘야 한다.

<br/><br/>

## Geometry Shader를 통해 Quad를 직육면체로 바꾸기

- Geometry Shader는 프로젝트의 에셋으로 생성되어야 한다.
    - Create > Shader > Unlit Shader를 통해 셰이더 에셋을 생성한다.
        
        ![2](https://user-images.githubusercontent.com/90246317/170819197-c7502e51-79a0-49cb-b08f-60278c849cba.png){: .align-center}
        
- 셰이더 작성
    
    ```glsl
    Shader "Unlit/CuboidShader"
    {
        Properties
        {
            _Height("Height", float) = 5.0
            _TopColor("Top Color", Color) = (0.0, 0.0, 1.0, 1.0)
            _BottomColor("Bottom Color", Color) = (1.0, 0.0, 0.0, 1.0)
        }
        SubShader
        {
            Tags
            {
                "RenderType"="Opaque"
            }
    
            Pass
            {
                HLSLPROGRAM
                #pragma vertex vert
                #pragma fragment frag
                #pragma geometry geom
    
                #include "UnityCG.cginc"
    
                uniform float _Height;
                uniform float4 _TopColor, _BottomColor;
    
                struct appdata
                {
                    float4 vertex : POSITION;
                    float2 uv : TEXCOORD0;
                };
    
                struct v2g
                {
                    float4 vertex : SV_POSITION;
                };
    
                struct g2f
                {
                    float4 vertex : SV_POSITION;
                    float4 color : COLOR;
                };
    
                v2g vert(appdata v)
                {
                    v2g o;
                    o.vertex = v.vertex;
                    return o;
                }
    
                [maxvertexcount(18)]
                void geom(triangle v2g input[3], inout TriangleStream<g2f> outStream)
                {		
                    g2f o[6];
                    float4 offset = float4(0.0f, 0.0f, -_Height, 0.0f);
    
                    for (int i = 0; i < 3; i++)
                    {		
                        o[i].vertex = UnityObjectToClipPos(input[i].vertex);
                        o[i].color = _BottomColor;
                    }
    
                    for (int i = 3; i < 6; i++)
                    {
                        o[i].vertex = input[i - 3].vertex + offset;
                        o[i].vertex = UnityObjectToClipPos(o[i].vertex);
                        o[i].color = _TopColor;
                    }
    
                    // bottom
                    outStream.Append(o[0]);
                    outStream.Append(o[2]);
                    outStream.Append(o[1]);
                    outStream.RestartStrip();
    
                    // top
                    outStream.Append(o[3]);
                    outStream.Append(o[4]);
                    outStream.Append(o[5]);
                    outStream.RestartStrip();
    
                    // side 1
                    outStream.Append(o[0]);
                    outStream.Append(o[3]);
                    outStream.Append(o[5]);
                    outStream.RestartStrip();
    
                    // side 2
                    outStream.Append(o[0]);
                    outStream.Append(o[5]);
                    outStream.Append(o[2]);
                    outStream.RestartStrip();
    
                    // side 3
                    outStream.Append(o[2]);
                    outStream.Append(o[5]);
                    outStream.Append(o[4]);
                    outStream.RestartStrip();
    
                    // side 4
                    outStream.Append(o[2]);
                    outStream.Append(o[4]);
                    outStream.Append(o[1]);
                    outStream.RestartStrip();
                }
    
                fixed4 frag(g2f i) : SV_Target
                {
                    return i.color;
                }
                ENDHLSL
            }
        }
    }
    ```
    

- 머티리얼 프로퍼티 선언
    - 머리티얼의 프로퍼티를 선언한다. “”로 감싼 문자열은 인스펙터에서 보여지는 이름이며, 뒤에 오는 것은 프로퍼티의 타입이다.
    - 직육면체(삼각기둥)의 키(_Height), 직육면체 상단부 색상(_TopColor)과 하단부 색상(_BottomColor)을 프로퍼티로 선언해주고 디폴트값을 지정해준다.
    
    ```glsl
    Shader "Unlit/CuboidShader"
    {
        Properties
        {
            _Height("Height", float) = 5.0
            _TopColor("Top Color", Color) = (0.0, 0.0, 1.0, 1.0)
            _BottomColor("Bottom Color", Color) = (1.0, 0.0, 0.0, 1.0)
        }
    ```
    

- SubShader의 태그 지정
    - RenderType은 Opaque로 지정해준다. Opaque는 대부분의 불투명한 셰이더에서 사용되는 렌더 타입이다.
    
    ```glsl
    SubShader
    {
        Tags
        {
            "RenderType"="Opaque"
        }
    ```
    

- 셰이더 프로그램 작성
    - HLSLPROGRAM과 ENDHLSL 사이에 HLSL 코드를 작성한다.
    - #pragma geometry geom 구문을 통해 컴파일러에게 지오메트리 셰이더를 사용할 것이며 그 메인 함수는 geom임을 알린다.
    - uniform 키워드를 통해 Properties에서 선언했던 프로퍼티들의 값을 가져온다. 이 때, 선언했던 프로퍼티와 동일한 이름을 사용해야 한다.
    
    ```glsl
    Pass
    {
        HLSLPROGRAM
        #pragma vertex vert
        #pragma fragment frag
        #pragma geometry geom

        #include "UnityCG.cginc"

        uniform float _Height;
        uniform float4 _TopColor, _BottomColor;
    ```
    

- vert, frag, geom 함수에서 입출력 타입으로 쓸 구조체들을 선언한다.
    - v2g: vert에서 출력되어 geom의 입력으로 들어가는 타입
    - g2f: geom에서 출력되어 frag의 입력으로 들어가는 타입
        - 컬러 또한 geom에서 바꿔줄 것이므로 관련 변수도 선언해준다.
    
    ```glsl
    struct appdata
    {
        float4 vertex : POSITION;
        float2 uv : TEXCOORD0;
    };

    struct v2g
    {
        float4 vertex : SV_POSITION;
    };

    struct g2f
    {
        float4 vertex : SV_POSITION;
        float4 color : COLOR;
    };
    ```
    

- Vertex Shader는 받은 정점 정보를 아무런 변화 없이 그대로 출력시킨다. vert에서 UnityObjectToClipPos를 호출해 정점의 좌표를 투영시켜 버리면 geom에서 정점의 계산 작업이 복잡해지므로, 투영은 이후 Geometry Shader에서 진행한다.
    
    ```glsl
    v2g vert(appdata v)
    {
        v2g o;
        o.vertex = v.vertex;
        return o;
    }
    ```
    

- geom을 정의할 때 maxvertexcount 어트리뷰트로 geom이 반환하는 정점의 최대 개수를 정의해줘야 한다. 여기서는 18개인데, 그 이유는 나중에 설명하겠다.
- geom의 파라미터는 Geometry Shader의 입출력을 담당한다.
    - 입력: 어떤 형태의 도형을 입력으로 받을지를 정의해주는 키워드와 배열 input으로 이루어져 있다. 여기서는 triangle 키워드를 사용했는데 기본 도형을 삼각형으로 정의하고 그 단위로 함수를 실행하기 위해서이다. 이 키워드에 따라 input 배열의 요소 수를 맞춰줘야 한다.
        
        
        | 키워드 | 배열 요소 수 | 의미 |
        | --- | --- | --- |
        | point | 1 | 정점 |
        | line | 2 | 단일 라인 |
        | triangle | 3 | 단일 삼각형 |
        | lineadj | 4 | 단일 라인 + 인접한 정점 2개 |
        | triangleadj | 6 | 단일  |
    - 출력:  출력 스트림을 선언하는데, 출력할 기본 도형에 따라 세 가지로 선언할 수 있다. 이 스트림의 Append 메서드를 호출하는 식으로 도형을 출력한다.
        
        
        | 타입 | 의미 |
        | --- | --- |
        | PointStream | 정점 단위로 출력 |
        | LineStream | 선 단위로 출력 |
        | TriangleStream | 삼각형 단위로 출력 |
    
    ```glsl
    [maxvertexcount(18)]
    void geom(triangle v2g input[3], inout TriangleStream<g2f> outStream)
    {		
    ```
    

- 함수를 구현하기 전 생각해야 할 것은, Quad를 직육면체로 만드려면 삼각형을 삼각기둥으로 만들어주면 된다는 사실이다. 두 삼각기둥이 붙은 면은 만들 필요가 없으므로, 하나의 삼각형 입력 당 6개의 삼각형을 출력해주면 된다. 그러므로 출력되는 정점은 18개인 것이다.
    
    ![3](https://user-images.githubusercontent.com/90246317/170819207-b8b28a37-ca7a-4cf5-a4e8-1c91396183f2.png){: .align-center}


- outStream에 Append해 삼각형을 만들 정점 정보들의 배열(g2f 타입 요소를 갖는) o를 선언한다.
- 그 다음 for문을 돌며 input으로 들어온 정점의 좌표를 o에 채워주되, UnityObjectToClipPos를 호출해 투영 과정을 거친다.
- o[0]~o[2]에는 input으로 들어온 정점들이 저장되며, 이들은 바닥의 삼각형을 이루게 된다. 그러므로 color에는 _BottomColor를 대입해준다.
- o[3]~[5]는 삼각기둥 윗면의 삼각형을 이룰 정점들인데, _Height 프로퍼티로 받아온 높이값을 이용해 설정한 offset을 좌표에 더해 준다.
    - offset의 z값에 -_Height가 들어간 이유는, 유니티는 왼손 좌표계를 사용하므로 기둥이 앞쪽으로 튀어나오게 하려면 z값이 음수가 되어야 하기 때문이다.
    - o[i].vertex의 정보를 채울 때 input[i-3].vertex를 사용하는 것은 위 그림처럼 정점들의 인덱스를 맞춰주기 위해서이다.
    - 역시 UnityObjectToClipPos를 호출해 정점을 투영해준 다음 그 값을 o[i].vertex에 저장하고, color는 _TopColor로 지정해준다.
    
    ```glsl
    g2f o[6];
    float4 offset = float4(0.0f, 0.0f, -_Height, 0.0f);

    for (int i = 0; i < 3; i++)
    {		
        o[i].vertex = UnityObjectToClipPos(input[i].vertex);
        o[i].color = _BottomColor;
    }

    for (int i = 3; i < 6; i++)
    {
        o[i].vertex = input[i - 3].vertex + offset;
        o[i].vertex = UnityObjectToClipPos(o[i].vertex);
        o[i].color = _TopColor;
    }
    
    ```
    

- outStream에 정점들을 Append하며 삼각형 정보를 출력한다.
    - 주의할 점은, TriangleStreamd은 triangle strip 방식으로 삼각형을 구성한다는 것이다.
    - 예를 들어, 정점 v1부터 정점 v7까지의 정보를 차례대로 Append할 경우, 다음과 같은 모습으로 삼각형이 구성된다. 즉 첫번째 삼각형까지는 처음 세 개의 정점 정보를 사용해 구성하되, 두번째 삼각형부터는 이전 두 개의 정점과 새로운 하나의 정점을 사용해 구성하는 것이다.
        
        ![4](https://user-images.githubusercontent.com/90246317/170819222-dc71f062-7dc4-4d2d-ac5d-d50dd1a0f96d.png){: .align-center}
        
        - 그러나 triangle strip 방식을 사용하면 노말 방향이 꼬이게 된다. 왼손 좌표계에서 삼각형은 정점을 시계방향으로 돌며 구성되어야 앞면이 된다. 위와 같이 삼각형들이 구성되면 삼각형들의 노말 방향이 뒤바뀌어 번갈아가며 백페이스 컬링이 발생한다.
    - 그러므로 여기서는 삼각형을 하나 구성할 때마다 RestartStrip을 호출해 TriangleStream의 strip 정보를 리셋해줘야 한다.
- 이제 삼각형의 앞면과 뒷면을 고려해 적절한 방향으로 정점을 Append하며 삼각형을 구성해준다.
    
    ```glsl
        // bottom
        outStream.Append(o[0]);
        outStream.Append(o[2]);
        outStream.Append(o[1]);
        outStream.RestartStrip();

        // top
        outStream.Append(o[3]);
        outStream.Append(o[4]);
        outStream.Append(o[5]);
        outStream.RestartStrip();

        // side 1
        outStream.Append(o[0]);
        outStream.Append(o[3]);
        outStream.Append(o[5]);
        outStream.RestartStrip();

        // side 2
        outStream.Append(o[0]);
        outStream.Append(o[5]);
        outStream.Append(o[2]);
        outStream.RestartStrip();

        // side 3
        outStream.Append(o[2]);
        outStream.Append(o[5]);
        outStream.Append(o[4]);
        outStream.RestartStrip();

        // side 4
        outStream.Append(o[2]);
        outStream.Append(o[4]);
        outStream.Append(o[1]);
        outStream.RestartStrip();
    }
    ```
    

![5](https://user-images.githubusercontent.com/90246317/170819237-abd09dcf-6aaa-4c1a-b0b8-11e200e014d7.png){: .align-center}

- Fragment Shader는 별다른 처리 없이 정점의 컬러를 그대로 반환해준다.
    
    ```glsl
                fixed4 frag(g2f i) : SV_Target
                {
                    return i.color;
                }
                ENDHLSL
            }
        }
    }
    ```
    

- 이제 셰이더가 완성되었으니, 이것을 새 머티리얼에 지정해준다.
    
    ![6](https://user-images.githubusercontent.com/90246317/170819247-3a565403-ba0d-453a-ab5c-711162d7918c.png){: .align-center}
    

- 그리고 Quad 오브젝트의 머티리얼을 이것으로 교체해 주면, 다음과 같은 직육면체가 완성된다!
    
    ![7](https://user-images.githubusercontent.com/90246317/170819254-8c848d6c-a4e3-4a43-acb9-01f361ba1e5f.png){: .align-center}
    

- Height의 변화에 따라 달라지는 직육면체의 모습이다.
    
    ![8)](https://user-images.githubusercontent.com/90246317/170819262-db001fc2-2b05-4d22-8dfc-3b41e85346a6.gif){: .align-center}


<br/><br/>

## 참고자료

- [Unity - Manual: Writing shaders](https://docs.unity3d.com/Manual/shader-writing.html)
- [유니티 셰이더 기본 7 - 지오메트리 셰이더 : 네이버 블로그](https://blog.naver.com/PostView.naver?blogId=canny708&logNo=221549767701&parentCategoryNo=&categoryNo=&viewDate=&isShowPopularPosts=false&from=postView)
- [Triangle Strips - Win32 apps : Microsoft Docs](https://docs.microsoft.com/en-us/windows/win32/direct3d9/triangle-strips)