---
date: 2020-11-17 11:01:47
layout: post
title: "The Stencil Buffer of Unity3D"
subtitle: Outlining, polygon filling, mirror restricting and shadow volume.
description: This article explains Unity Stencil Buffer with outlining, polygon filling, mirror restricting and shadow volume.
image: https://res.cloudinary.com/dokdkay80/image/upload/c_scale,w_760/v1604503676/StencilBuffer/p1_fddhcm.png
optimized_image: https://res.cloudinary.com/dokdkay80/image/upload/c_scale,w_380/v1604503676/StencilBuffer/p1_fddhcm.png
category: Unity3D
tags:
- Shader
- Unity3D
author: liu_if_else
paginate: false
math: true
---

# Menu
- [The stencil buffer](#the-stencil-buffer)
  * [The stencil](#the-stencil)
  * [The stencil and depth buffer](#the-stencil-and-depth-buffer)
  * [Reading and writing](#reading-and-writing)
- [Outlining](#outlining)
  * [The shader](#the-shader)
  * [Explanation](#explanation)
  * [Screenshots from rendering](#screenshots-from-rendering)
- [Polygon Filling](#polygon-filling)
  * [The shader](#the-shader-1)
  * [Explanation](#explanation-1)
  * [Screenshots from rendering](#screenshots-from-rendering-1)
- [Mirror Restricting](#mirror-restricting)
  * [The shaders](#the-shaders)
  * [Explanation](#explanation-2)
  * [Screenshots from rendering](#screenshots-from-rendering-2)
- [Shadow Volume](#shadow-volume)
  * [The shader](#the-shader-2)
  * [Explanation](#explanation-3)
  * [Screenshots from rendering](#screenshots-from-rendering-3)


# The stencil buffer

## The stencil

A stencil is a flat material where parts are cutted out, often in shapes or patterns. It's a traditional tool used by printing and clothing industry. Through the cut-outs, you draw or paint colours on the surface of the priting material beneath the stencil. 
  
Imagining the computer screen as a x\*y matrix of zeros, the stencil buffer cuts those zeros out by sets parts of them to 1,2,3,..., and 255. In each pass of the shader, the final output colours are masked by the stencil test, in which the fragment's current stencil value compares to the pre-setting one. The pixels will be discarded if they haven't passed the test. In this way, the buffer works like a rectangle stencil on top of the pixels.
  
## The stencil and depth buffer
  
According to the Wikipedia, the very modern GPU architecture binds the stencil and the depth buffer together. In other words, the 24 bits depth and 8 bits stencil information of a single fragment could be in a continuous 32 bits area of the Graphics RAM. When you create a render-texture in Unity, setting RenderTexture.depth to 32 will enable the 8 bits stencil buffer while 16(and 24?) disable it. Perhaps because of the proximity, the Z-test result can be obtained in the stencil testing stage.  
  
## Reading and writing

Between the fragment shader and the Blending of the OpenGL pipeline, there are three testing stages: scissor testing, stencil testing and Z-testing. They share the same OpenGL state machine style syntax, which is all about pre-settings and keywords.

For stencil test, firstly you need to declar a Stencil struct in the SubShader or Pass scope, and then use "Ref" keyword to set a standard stencil value, "Comp" to set a comparing condition, "Pass" to define the furthur operation for the passed stencil comparetion, "Fail" for the failed one, and "ZFail" for the passed stencil test with the failed depth test. All the comparison condition and stencil operation keywords can be found at the <a href="https://docs.unity3d.com/Manual/SL-Stencil.html">Unity documentation</a>.

```c
Stencil {
    //the base value for the test to compare with
    Ref 0           //value's range: 0-255
    //compare condition
    Comp Equal     //default keyword: always(always pass)
    //the furthur operation if the comparetion successes
    Pass keep       //default keyword: keep(keep the fragment's current stencil value)
    //the furthur operation if the comparetion fails
    Fail keep       //default keyword: keep
    //the furthur operation if the stencil comparetion successes but Z-test fails
    ZFail IncrWrap  //default keyword: keep
}
```

# Outlining

## The shader

```c
Shader "Unlit/StentilOutline"
{
	Properties
	{
		_MainTex ("Texture", 2D) = "white" {}
	}
	SubShader
	{
		Tags { "RenderType"="Opaque" }
		LOD 100
		Stencil {
		     Ref 0          //0-255
		     Comp Equal     //default:always
		     Pass IncrSat   //default:keep
		     Fail keep      //default:keep
		     ZFail keep     //default:keep
		}

		Pass
		{
			CGPROGRAM
			#pragma vertex vert
			#pragma fragment frag
			// make fog work
			#pragma multi_compile_fog
			
			#include "UnityCG.cginc"

			struct appdata
			{
				float4 vertex : POSITION;
				float2 uv : TEXCOORD0;
			};

			struct v2f
			{
				float2 uv : TEXCOORD0;
				UNITY_FOG_COORDS(1)
				float4 vertex : SV_POSITION;
			};

			sampler2D _MainTex;
			float4 _MainTex_ST;
			
			v2f vert (appdata v)
			{
				v2f o;
				o.vertex = UnityObjectToClipPos(v.vertex);
				o.uv = TRANSFORM_TEX(v.uv, _MainTex);
				UNITY_TRANSFER_FOG(o,o.vertex);
				return o;
			}
			
			fixed4 frag (v2f i) : SV_Target
			{
				// sample the texture
				fixed4 col = tex2D(_MainTex, i.uv);
				// apply fog
				UNITY_APPLY_FOG(i.fogCoord, col);
				//return fixed4(1,1,0,1);
				return col;
			}
			ENDCG
		}

		Pass
		{
		    CGPROGRAM
		    #pragma vertex vert
		    #pragma fragment frag
		    // make fog work
		    #pragma multi_compile_fog
		    
		    #include "UnityCG.cginc"

		    struct appdata
		    {
		        float4 vertex : POSITION;
		        float4 normal: NORMAL;
		        float2 uv : TEXCOORD0;
		    };

		    struct v2f
		    {
		        float2 uv : TEXCOORD0;
		        UNITY_FOG_COORDS(1)
		        float4 vertex : SV_POSITION;
		    };

		    sampler2D _MainTex;
		    float4 _MainTex_ST;
		    
		    v2f vert (appdata v)
		    {
		        v2f o;
		        o.vertex=v.vertex+normalize(v.normal)*0.01f;
		        o.vertex = UnityObjectToClipPos(o.vertex);
		        o.uv = TRANSFORM_TEX(v.uv, _MainTex);
		        UNITY_TRANSFER_FOG(o,o.vertex);
		        return o;
		    }
		    
		    fixed4 frag (v2f i) : SV_Target
		    {
		        // sample the texture
		        fixed4 col = tex2D(_MainTex, i.uv);
		        // apply fog
		        UNITY_APPLY_FOG(i.fogCoord, col);
		        return fixed4(1,1,1,1);
		    }
		    ENDCG
		}
	}
}
```

## Explanation

```c
		Stencil {
		     Ref 0          //0-255
		     Comp Equal     //default:always
		     Pass IncrSat   //default:keep
		     Fail keep      //default:keep
		     ZFail keep     //default:keep
		}
```
This struct works in all passes since it is in the SubShader scope. Generally, all rasterized fragments will pass the "Ref 0" and "Comp Equal" test in the first shader pass since their stencil values are initial zero. "Pass IncrSat" adds one to the stencil value of each passed fragment. "Sat" makes the addition in the saturation style: if the result is higher then 255, it stays at 255.     

```c
...
		        o.vertex=v.vertex+normalize(v.normal)*0.01f;
...		       
```

In the second pass, at first, vertices are magnified alone their normal direction to draw a bigger image. Within the new rendering area, the pixels rendered by the previous pass are with stenticl value of one. They can not pass the "Ref 0" and "Comp Equal" test and therefore will be discarded. On the other hand, the fragments of the newly inflated area will succeed the test since their stencil values are still zero.

```c
...
                return fixed4(1,1,1,1);
...
```

Color these fragments and you will get the rim.

## Screenshots from rendering

| ![placeholder](https://res.cloudinary.com/dokdkay80/image/upload/v1604503677/StencilBuffer/p6_e26gjj.png) | 
|:--:| 
| <span style="color: gray; font-size: 80%">Picture 1: Geometries with StencilPerPassOutline.shader from the demo project</span> |

| ![placeholder](https://res.cloudinary.com/dokdkay80/image/upload/v1604503680/StencilBuffer/p3_ywdz2z.png) | 
|:--:| 
|<span style="color: gray; font-size: 80%">Picture 2: Humanoid with StencilOutline.shader from the demo project</span>|


# Polygon Filling

## The shader

```c
Shader "Unlit/PolygonsBeta"
{
    Properties
    {
        _MainTex ("Texture", 2D) = "white" {}
    }
    SubShader
    {
        Tags { "RenderType"="Opaque" }
        LOD 100

        CGINCLUDE
        #include "UnityCG.cginc"
        struct appdata
        {
            float4 vertex : POSITION;
            float2 uv : TEXCOORD0;
        };

        struct v2f
        {
            float2 uv : TEXCOORD0;
            UNITY_FOG_COORDS(1)
            float4 vertex : SV_POSITION;
        };

        sampler2D _MainTex;
        float4 _MainTex_ST;
        
        v2f vert (appdata v)
        {
            v2f o;
            o.vertex = UnityObjectToClipPos(v.vertex);
            o.uv = TRANSFORM_TEX(v.uv, _MainTex);
            UNITY_TRANSFER_FOG(o,o.vertex);
            return o;
        }
        ENDCG

        Pass
        {
            Stencil {
                Ref 0           //0-255
                Comp always     //default:always
                Pass IncrWrap   //default:keep
                Fail keep       //default:keep
                ZFail IncrWrap  //default:keep
            }

            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            // make fog work
            #pragma multi_compile_fog

            fixed4 frag (v2f i) : SV_Target
            {
                // sample the texture
                fixed4 col = tex2D(_MainTex, i.uv);
                // apply fog
                UNITY_APPLY_FOG(i.fogCoord, col);
                return fixed4(0,0,0,0);
            }
            ENDCG
        }
        
        Pass
        {
            Stencil {
                Ref 2          //0-255
                Comp Equal     //default:always
                Pass keep      //default:keep
                Fail keep      //default:keep
                ZFail keep     //default:keep
            }
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            // make fog work
            #pragma multi_compile_fog
            
            #include "UnityCG.cginc"

            fixed4 frag (v2f i) : SV_Target
            {
                // sample the texture
                fixed4 col = tex2D(_MainTex, i.uv);
                // apply fog
                UNITY_APPLY_FOG(i.fogCoord, col);
                return fixed4(0.2,0.2,0.2,1);
            }
            ENDCG
        }

        Pass
        {
            Stencil {
                Ref 3          //0-255
                Comp equal     //default:always
                Pass keep      //default:keep
                Fail keep      //default:keep
                ZFail keep     //default:keep
            }
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            // make fog work
            #pragma multi_compile_fog
            
            #include "UnityCG.cginc"

            fixed4 frag (v2f i) : SV_Target
            {
                // sample the texture
                fixed4 col = tex2D(_MainTex, i.uv);
                // apply fog
                UNITY_APPLY_FOG(i.fogCoord, col);
                return fixed4(0.6,0.6,0.6,1);
            }
            ENDCG
        }

        Pass
        {
            Stencil {
                Ref 4          //0-255
                Comp equal     //default:always
                Pass keep      //default:keep
                Fail keep      //default:keep
                ZFail keep     //default:keep
            }
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            // make fog work
            #pragma multi_compile_fog
            
            #include "UnityCG.cginc"

            fixed4 frag (v2f i) : SV_Target
            {
                // sample the texture
                fixed4 col = tex2D(_MainTex, i.uv);
                // apply fog
                UNITY_APPLY_FOG(i.fogCoord, col);
                return fixed4(1,1,1,1);
            }
            ENDCG
        }
    }
}
```

## Explanation

This shader is similar to the example shader of "Red/Green/Blue" on the <a href="https://docs.unity3d.com/Manual/SL-Stencil.html">Unity website</a>. Both shaders rely on the stencil buffer to differenciate the cross areas of the geometries. 

Each pass has its own stencil struct. In the first pass all fragments succeed the test and add one to their stencil values. In this pass all overlapped areas are marked. Their stencil values are equal to the number of the overlapped geometries. The next three passes render these areas.  

## Screenshots from rendering
  
| ![placeholder](https://res.cloudinary.com/dokdkay80/image/upload/v1604503676/StencilBuffer/p1_fddhcm.png) | 
|:--:| 
| <span style="color: gray; font-size: 80%">Picture 3: Geometries with PolygonsBeta.shader from the demo project</span> |

| ![placeholder](https://res.cloudinary.com/dokdkay80/image/upload/v1604503676/StencilBuffer/p2_zkey5s.png) | 
|:--:| 
| <span style="color: gray; font-size: 80%">Picture 4: Geometries with polygons.shader from the demo project.<br/>The code of the Archimedean spiral is from my <a href="https://blog.csdn.net/liu_if_else/article/details/51458603">previous article(Chinese)</a>. </span> |


# Mirror Restricting

## The shaders

```c
Shader "Unlit/TwoPassReflection"
{
	Properties
	{
		_MainTex ("Texture", 2D) = "white" {}
	}
	SubShader
	{
		Tags { "RenderType"="Opaque" "Queue"="Geometry" }
		LOD 100

		Pass
		{
			CGPROGRAM
			#pragma vertex vert
			#pragma fragment frag
			// make fog work
			#pragma multi_compile_fog
			
			#include "UnityCG.cginc"

			struct appdata
			{
				float4 vertex : POSITION;
				float2 uv : TEXCOORD0;
			};

			struct v2f
			{
				float2 uv : TEXCOORD0;
				UNITY_FOG_COORDS(1)
				float4 vertex : SV_POSITION;
			};

			sampler2D _MainTex;
			float4 _MainTex_ST;
			
			v2f vert (appdata v)
			{
				v2f o;
				o.vertex = UnityObjectToClipPos(v.vertex);
				o.uv = TRANSFORM_TEX(v.uv, _MainTex);
				UNITY_TRANSFER_FOG(o,o.vertex);
				return o;
			}
			
			fixed4 frag (v2f i) : SV_Target
			{
				// sample the texture
				fixed4 col = tex2D(_MainTex, i.uv);
				// apply fog
				UNITY_APPLY_FOG(i.fogCoord, col);
				return col;
			}
			ENDCG
		}

 		Pass
 		{
 		    Stencil {
 		        Ref 1          //0-255
 		        Comp Equal     //default:always
 		        Pass keep      //default:keep
 		        Fail keep      //default:keep
 		        ZFail keep     //default:keep
 		    }
 		    ZTest Always
 		    CGPROGRAM
 		    #pragma vertex vert
 		    #pragma fragment frag
 		    // make fog work
 		    #pragma multi_compile_fog
 		    
 		    #include "UnityCG.cginc"

 		    struct appdata
 		    {
 		        float4 vertex : POSITION;
 		        float2 uv : TEXCOORD0;
 		        float4 normal: NORMAL;
 		    };

 		    struct v2f
 		    {
 		        float2 uv : TEXCOORD0;
 		        UNITY_FOG_COORDS(1)
 		        float4 vertex : SV_POSITION;
 		    };

 		    sampler2D _MainTex;
 		    float4 _MainTex_ST;
 		    
 		    v2f vert (appdata v)
 		    {
 		        v2f o;
 		        v.vertex.xyz=reflect(v.vertex.xyz,float3(-1.0f,0.0f,0.0f));
 		        v.vertex.xyz=reflect(v.vertex.xyz,float3(0.0f,1.0f,0.0f));
 		        v.vertex.x+=1.5f;
 		        o.vertex = UnityObjectToClipPos(v.vertex);
 		        o.uv = TRANSFORM_TEX(v.uv, _MainTex);
 		        UNITY_TRANSFER_FOG(o,o.vertex);
 		        return o;
 		    }
 		    
 		    fixed4 frag (v2f i) : SV_Target
 		    {
 		        // sample the texture
 		        fixed4 col = tex2D(_MainTex, i.uv);
 		        // apply fog
 		        UNITY_APPLY_FOG(i.fogCoord, col);
 		        return col;
 		    }
 		    ENDCG
 		}
	}
}
```
```c
Shader "Unlit/Mirror"
{
	Properties
	{
		_MainTex ("Texture", 2D) = "white" {}
	}
	SubShader
	{
		Tags { "RenderType"="Opaque" "Queue"="Geometry-1" } //you have to render the mirror first
		LOD 100

 		Stencil {
 		    Ref 0          //0-255
 		    Comp always    //default:always
 		    Pass IncrSat   //default:keep
 		    Fail keep      //default:keep
 		    ZFail keep     //default:keep
 		}

		Pass
		{
			CGPROGRAM
			#pragma vertex vert
			#pragma fragment frag
			// make fog work
			#pragma multi_compile_fog
			
			#include "UnityCG.cginc"

			struct appdata
			{
				float4 vertex : POSITION;
				float2 uv : TEXCOORD0;
			};

			struct v2f
			{
				float2 uv : TEXCOORD0;
				UNITY_FOG_COORDS(1)
				float4 vertex : SV_POSITION;
			};

			sampler2D _MainTex;
			float4 _MainTex_ST;
			
			v2f vert (appdata v)
			{
				v2f o;
				o.vertex = UnityObjectToClipPos(v.vertex);
				o.uv = TRANSFORM_TEX(v.uv, _MainTex);
				UNITY_TRANSFER_FOG(o,o.vertex);
				return o;
			}
			
			fixed4 frag (v2f i) : SV_Target
			{
				// sample the texture
				fixed4 col = tex2D(_MainTex, i.uv);
				// apply fog
				UNITY_APPLY_FOG(i.fogCoord, col);
				return fixed4(0.2f,0.2f,0.2f,1.0f);
			}
			ENDCG
		}
	}
}
```
## Explanation

The Mirror shader assists the TwoPassReflection shader to simulate a simple mirror effect. In the TwoPassReflection shader, the vertices are processed normally in the first pass and reflected in the second. The reflecting area often overlaps the mirror object therefore the ZTest of the second pass must set to Always. 

If you insert a flat plane between those reflected fragments, visually it becomes a mirror. However, the effect will be debunked if the mirror object failed to cover the whole reflection(see picture 5 beneath).

The solution is to render the mirror first, marking its fragments by setting their stencil values to one, and then to add a stencil test to the TwoPassReflection shader's second pass, discarding the fragments outside the mirror area.
  
## Screenshots from rendering

| ![placeholder](https://res.cloudinary.com/dokdkay80/image/upload/v1604503677/StencilBuffer/p7_lxi6i9.png) | 
|:--:| 
| <span style="color: gray; font-size: 80%; text-align: center">Picture 5: Goblin with TwoPassReflection.shader(without the stencil test)</span> |

| ![placeholder](https://res.cloudinary.com/dokdkay80/image/upload/v1604503678/StencilBuffer/p8_fmquxq.png) | 
|:--:| 
| <span style="color: gray; font-size: 80%; text-align: center">Picture 6: Goblin with TwoPassReflection.shader and Plane with Mirror.shader</span>|

# Shadow Volume

## The shader

```c
Shader "Unlit/SV_DepthFailBeta"
{
    Properties
    {
    }
    SubShader
    {
        Tags { "RenderType"="Opaque" "Queue"="Geometry+1"}  //delay the shadow geometry rendering
        LOD 100
        
        CGINCLUDE       
        #include "UnityCG.cginc"
        struct appdata
        {
            float4 vertex : POSITION;
        };

        struct v2f
        {
            UNITY_FOG_COORDS(1)
            float4 vertex : SV_POSITION;
        };

        v2f vert (appdata v)//three passes share the same vertex and fragment shaders
        {
            v2f o;
            o.vertex = UnityObjectToClipPos(v.vertex);
            UNITY_TRANSFER_FOG(o,o.vertex);
            return o;
        }
        
        fixed4 frag (v2f i) : SV_Target
        {
            // apply fog
            UNITY_APPLY_FOG(i.fogCoord, col);
            return fixed4(0.3,0.3,0.3,1);           //the color of the shadow
        }
        ENDCG

        Pass
        {
            Cull Front          //first pass renders the inside surface
            Stencil {           
                Ref 0           //0-255
                Comp always     //default:always
                Pass keep       //default:keep
                Fail keep       //default:keep
                ZFail IncrWrap  //default:keep
            }

            ColorMask 0         //don't have to touch the color buffer
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            // make fog work
            #pragma multi_compile_fog
            ENDCG
        }
        
        Pass
        {
            Cull Back           //second pass renders the outside surface
            Stencil {
                Ref 0           //0-255
                Comp always     //default:always
                Pass keep       //default:keep
                Fail keep       //default:keep
                ZFail DecrWrap  //default:keep
            }
            ColorMask 0         //no writing to the color buffer
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            // make fog work
            #pragma multi_compile_fog
            ENDCG
        }

        Pass
        {
            Cull Back          //identify the shadow fragments
            Stencil {
                Ref 1          //0-255
                Comp equal     //default:always
                Pass keep      //default:keep
                Fail keep      //default:keep
                ZFail keep     //default:keep
            }
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            // make fog work
            #pragma multi_compile_fog
            ENDCG
        }
    }
}
```

## Explanation

The idea of shadow volume is to instaniate the shadow as a geometry, creating a mesh equals to the shadow's shape(see the cylinder shadow in picture 7) and then to identify all shadowed fragments along with the rendering process of this geometry.

In the rendering process, there are several algorithms to derterminate the shadowed fragments. The demo shader uses "Depth Fail" method, which also names as "Carmack’s reverse". From the perspective of stencil buffer, I simplify this algorithm to three steps:

1, All shadowable objects, like ground, trees, rocks and etc., render first the shadow polyhedron after. The first shader pass of the shadow polyhedron is set to "cull front", rendering the inside surface only. The stencil test of the pass marks the Z-test-failed fragments through increasing their stencil value. The marked fragments could been inside or outside the shadow geometry. The fragments inside the shadow geometry are what we are looking for. The fragments outside the shadow geometry(e.g, between the camera and the shadow polyhedron) will be excluded in step 2. 

2, Set "cull back", the second pass renders the outside surface of the shadow geometry. The z-test-failed check finds shadow fragments outside the shadow polyhedron and then "ZFail DecrWrap" operation decreases their stencil values by one.

3, Consequently, only the fragments of the shadowable objects inside the shadow geometry are with the stencil value of 1 at the final step, rendering these pixels to get the correct shadow.

From theory to practice, the three steps above have also explained the three passes of the shader SV_DepthFailBeta.  

Actually, this single shader is not sufficient to implement the shadow volume technique. For instance, the appropriate shadow geometry, imported mesh or generated in the runtime, is a must but the demo has faked it through using the Unity standard primitive. Moreover, some details such as other shadows in the shadow polyhedron are not included in this discussion yet. Nonetheless, the shader illustrates well the stencil buffer's important role of putting this rendering technique into effect.
  
## Screenshots from rendering

| ![placeholder](https://res.cloudinary.com/dokdkay80/image/upload/v1604503678/StencilBuffer/p5_ts0u94.png) | 
|:--:| 
|   <span style="color: gray; font-size: 80%; text-align: center">Picture 7: Geometries with SV_DepthFailBeta.shader and the cylinder shadow</span> |

| ![placeholder](https://res.cloudinary.com/dokdkay80/image/upload/v1604503678/StencilBuffer/p4_lf3jzw.png) | 
|:--:| 
|  <span style="color: gray; font-size: 80%; text-align: center">Picture 8: Geometries with SV_DepthFailBeta.shader from the demo project</span> |

-----
**Github Link of the demo project:**   
<a href="https://github.com/liu-if-else/UnityStencilBufferUses">https://github.com/liu-if-else/UnityStencilBufferUses</a>

-----
**References:**

Shadow Volume–Depth Fail - wikipedia  
<a href="https://en.wikipedia.org/wiki/Shadow_volume#Depth_fail">https://en.wikipedia.org/wiki/Shadow_volume#Depth_fail</a>
  
Creating Reflections and Shadows Using Stencil - BuffersMark J. Kilgard  
<a href="https://www2.cs.duke.edu/courses/spring15/cps124/classwork/14_buffers/stencil.pdf">https://www2.cs.duke.edu/courses/spring15/cps124/classwork/14_buffers/stencil.pdf</a>
  
Stencil Shadow Volume - OGL  
<a href="http://ogldev.atspace.co.uk/www/tutorial40/tutorial40.html">http://ogldev.atspace.co.uk/www/tutorial40/tutorial40.html</a>
  
ShaderLab: Stencil - Unity Technology  
<a href=" https://docs.unity3d.com/Manual/SL-Stencil.html"> https://docs.unity3d.com/Manual/SL-Stencil.html</a>

Simon F’s answer to topic of ‘Uses for Stencil Buffer’ - Simon F  
<a href=" https://computergraphics.stackexchange.com/questions/5046/uses-for-stencil-buffer"> https://computergraphics.stackexchange.com/questions/5046/uses-for-stencil-buffer</a>

----
**The Chinese version of this articel:**  
<a href="https://blog.csdn.net/liu_if_else/article/details/86316361
">https://blog.csdn.net/liu_if_else/article/details/86316361</a>

