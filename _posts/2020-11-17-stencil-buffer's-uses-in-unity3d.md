---
date: 2020-11-17 11:01:47
layout: post
title: "Stencil Buffer's Uses in Unity3D"
subtitle: Outlining, polygon filling, mirror restricting and shadow volume.
description: Demonstrate the uses of Unity Stencil Buffer with outlining, polygon filling, mirror restricting and shadow volume.
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
  * [Stencil and depth buffer](#stencil-and-depth-buffer)
  * [Reading and writing](#reading-and-writing)
- [Outlining](#outlining)
  * [The shader](#the-shader)
  * [Remarks](#remarks)
  * [The resulting screenshots](#the-resulting-screenshots)
- [Polygon Filling](#polygon-filling)
  * [The shader](#the-shader-1)
  * [Remarks](#remarks-1)
  * [The resulting screenshots](#the-resulting-screenshots-1)
- [Mirror Restricting](#mirror-restricting)
  * [The shaders](#the-shaders)
  * [Remarks](#remarks-2)
  * [The resulting screenshots](#the-resulting-screenshots-2)
- [Shadow Volume](#shadow-volume)
  * [The shader](#the-shader-2)
  * [Remarks](#remarks-3)
  * [The resulting screenshots](#the-resulting-screenshots-3)


# The stencil buffer

## The stencil

A stencil is a flat material where parts, usually shapes or patterns, have been cutted out. It's a common and traditional tool used by many industries such as printing, clothing, painting and etc. Through those holes, you draw or paint the desired images on the surface below the stencil. 
  
If you imagine the computer screen as a x\*y matrix of 0, the stencil buffer decides the shape of the holes by setting parts of those zeros to 1,2,3,..., and 255. In each pass of the shader, all the operations are filtered by the stencil test, in which the fragment's stencil value compares to the pre-setting one. The pixels will be discarded if they haven't passed the test. In this way, the stencil buffer becomes a mask and the render texture the paper under it.   
  
## Stencil and depth buffer
  
According to the Wikipedia, the very modern GPU architecture binds the stencil and depth buffer together. For instance, in a continuous 32 bits area of the Graphics RAM, there could be 24 bits for depth buffer and 8 bits for stencil buffer, both belonged to the same fragment. When you create a render-texture in Unity, setting RenderTexture.depth to 32 will enable a 8 bits stencil buffer while 16(and 24?) will disable it. Maybe because of they are so close together, you can actually get the Z-test result in the stencil testing(or Z-test modifies the stencil value?).  
  
## Reading and writing

In the OpenGL pipeline, between the fragment shader and the Blending, there are three testing stages that are scissor testing, stencil testing and Z-testing. All these testing operations have the OpenGL state machine style syntax, which is all about pre-setting and keywords.

For stencil test, firstly you need to declar a Stencil struct in SubShader or Pass scope, and then use "Ref" to set a base stencil value to compare, "Comp" to set a comparing condition, "Pass" to determine the furthur operation after the passed comparetion, "Fail" after the failed one, and "ZFail" after the passed stencil test combining the failed depth test. All the comparison functions and stencil operation keywords can be found at <a href="https://docs.unity3d.com/Manual/SL-Stencil.html">Unity documentation</a>.

```c
Stencil {
    //the base value to compare with
    Ref 0           //value's range: 0-255
    //compare condition
    Comp Equal     //default keyword: always(always pass)
    //the furthur operation if the comparetion successes
    Pass keep       //default keyword: keep(keep the fragment's current stencil value)
    //the furthur operation if the comparetion fails
    Fail keep       //default keyword: keep
    //if the stencil comparetion successes but Z-test fails
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

## Remarks

```c
		Stencil {
		     Ref 0          //0-255
		     Comp Equal     //default:always
		     Pass IncrSat   //default:keep
		     Fail keep      //default:keep
		     ZFail keep     //default:keep
		}
```
The struct has writen in the SubShader scope therefore it restricts all passes. In the first pass, under the ideal condition, all rasterized fragments will pass the "Ref 0" and "Comp Equal" test since zero is the initial stencil value. Then "Pass IncrSat" will add one to the stencil value of passed fragments. "Sat" makes the addition in the saturation style which means that if the value were the highest 255 then it would stay 255.     

```c
...
		        o.vertex=v.vertex+normalize(v.normal)*0.01f;
...		       
```

In the second pass, verteces will be magnified alone their normal direction at first. Most of the later rasterized fragments would be discarded since their stencil values, added by the previous pass, could not pass the "Ref 0" and "Comp Equal" test. On the other hand, the stencil values of the fragments in the newly inflated area should still be the initial value of zero thus they would succeed the test.  

```c
...
                return fixed4(1,1,1,1);
...
```

Color these fragments and you will get the rim.

## The resulting screenshots

| ![placeholder](https://res.cloudinary.com/dokdkay80/image/upload/v1604503677/StencilBuffer/p6_e26gjj.png) | 
|:--:| 
| <span style="color: gray; font-size: 80%">Picture 1: Using the StencilPerPassOutline.shader in the demo project</span> |

| ![placeholder](https://res.cloudinary.com/dokdkay80/image/upload/v1604503680/StencilBuffer/p3_ywdz2z.png) | 
|:--:| 
|<span style="color: gray; font-size: 80%">Picture 2: Using the StencilOutline.shader in the demo project</span>|


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

## Remarks

This shader is similar to the example shader of "Red/Green/Blue" on the <a href="https://docs.unity3d.com/Manual/SL-Stencil.html">Unity website</a>, both relying on the stencil buffer to differenciate the cross areas of the geometries. 

Each pass has its own stencil struct. In first pass all tests will be succeeded and then add one to the stencil value of the rendered fragments. The next three passes will only render the field marked by the stencil value of 2, 3 and 4.  

## The resulting screenshots
  
| ![placeholder](https://res.cloudinary.com/dokdkay80/image/upload/v1604503676/StencilBuffer/p1_fddhcm.png) | 
|:--:| 
| <span style="color: gray; font-size: 80%">Picture 3: Using the PolygonsBeta.shader in the demo project</span> |

| ![placeholder](https://res.cloudinary.com/dokdkay80/image/upload/v1604503676/StencilBuffer/p2_zkey5s.png) | 
|:--:| 
| <span style="color: gray; font-size: 80%">Picture 4: Using the polygons.shader in the demo project.<br/>The code of the Archimedean spiral comes from my <a href="https://blog.csdn.net/liu_if_else/article/details/51458603">previous article(Chinese)</a>. </span> |


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
## Remarks

The Mirror shader assists the TwoPassReflection shader to correctly and concisely simulate the mirroring effect. In the TwoPassReflection shader, the first pass processes all vertices normally and the second pass will reflect them by modifying their position. The ZTest of the second pass must set to Always in case of a mirror object has blocked the reflecting fragments. 

If you insert a flat plane between those reflected fragments, visually it becomes a mirror. However, this effect would be debunked if the mirror object failed to cover the whole reflection(see picture 5 beneath).

The solution is to add the stencil test to the TwoPassReflection shader's second pass, agreeing to only render the fragments with stencil value of one, which can only be marked by the Mirror shader. Hence the rendering queue here is tricky. Mirror must be rendered first.
  
## The resulting screenshots

| ![placeholder](https://res.cloudinary.com/dokdkay80/image/upload/v1604503677/StencilBuffer/p7_lxi6i9.png) | 
|:--:| 
| <span style="color: gray; font-size: 80%; text-align: center">Picture 5: Using the TwoPassReflection.shader(without stencil test)</span> |

| ![placeholder](https://res.cloudinary.com/dokdkay80/image/upload/v1604503678/StencilBuffer/p8_fmquxq.png) | 
|:--:| 
| <span style="color: gray; font-size: 80%; text-align: center">Picture 6: Using the TwoPassReflection.shader and the Mirror.shader</span>|

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
        
        CGINCLUDE       //three passes have the same vertex and fragment shaders
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

        v2f vert (appdata v)
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
            ColorMask 0         //no writing to color buffer
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

## Remarks

The idea of shadow volume is to instaniate the shadow as a geometry, or say object with mesh that equals to the shadow's shape(see the selected cylinder object in picture 7) and then to identify all the shadowed fragments along with rendering process of this geometry.

There are several algorithms to derterminate the shadowed fragments. The demo shader has used "Depth Fail", which also names as "Carmack’s reverse". From the perspective of stencil buffer, I have simplified it by three steps:

1, All common objects, like ground, trees and etc. should be rendered first and then the shadow polyhedron. The first pass of the shadow-renderer's shader will "cull front", rendering the inside surface of the shadow geometry. The stencil test of the first pass will increase the stencil value of the Z-test-failed fragments. Here could be two cases. The regular objects, or parts of them, inside the shadow geometry have blocked the inside surface of the shadow. These are the shadowed fragments that we are looking for. It's also possible that the Z-test failure were caused by the objects that are outside the shadow geometry(between the camera and the shadow polyhedron). These fragments should not be shadowed. However, their stencil values will still be marked, but just for now. We will solve it in step 2.

2, Setting "cull back", the second pass renders the outside surface of the shadow geometry. After the rasterization, the fragments can not be blocked by the objects inside the shadow geometry but can still be blocked by the objects outside. When these Z-test-failed fragments were found again, the "ZFail DecrWrap" operation will decrease their stencil values by one. 

3, Consequently, only the fragments of the common objects inside the shadow geometry have the stencil value of 1 at the final step. The correct shadow will be constructed by rendering these pixels. 

From theory to practice, the three steps above have also explained what happened in the shader's three passes.  

Actually, this single shader is not sufficient to implement shadow volume. For instance, the appropriate shadow geometry, imported mesh or generated in the runtime, is a must. The demo has faked it by the Unity standard primitive. Moreover, the details such as other shadows in the shadow polyhedron were not discussed yet. Nonetheless, the shader illustrates well the stencil buffer's important role of putting this rendering technique into effect.
  
## The resulting screenshots

| ![placeholder](https://res.cloudinary.com/dokdkay80/image/upload/v1604503678/StencilBuffer/p5_ts0u94.png) | 
|:--:| 
|   <span style="color: gray; font-size: 80%; text-align: center">Picture 7: Using the SV_DepthFailBeta.shader and selecting the cylinder shadow geometry</span> |

| ![placeholder](https://res.cloudinary.com/dokdkay80/image/upload/v1604503678/StencilBuffer/p4_lf3jzw.png) | 
|:--:| 
|  <span style="color: gray; font-size: 80%; text-align: center">Picture 8: Using the SV_DepthFailBeta.shader in the demo project</span> |

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

