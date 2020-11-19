---
date: 2020-11-04 13:37:54
layout: post
title: Unity3D Audio Visualizer
subtitle: Understand Unity3D's GetSpectrumData function and use it make a audio visulizer.
description: Understand the Unity GetSpectrumData function, the spectrum datas, the Fourier transform and related theorems and ultilize them to make a audio visualizer in Unity3D.
image: https://res.cloudinary.com/dokdkay80/image/upload/c_scale,w_760/v1604498366/AudioVisualizer/av1_itqijr.png
optimized_image: https://res.cloudinary.com/dokdkay80/image/upload/c_scale,w_380/v1604498366/AudioVisualizer/av1_itqijr.png
category: Unity3D
tags:
- Unity3D
- AudioVisualizer
author: liu_if_else
paginate: false
math: true
---
# Menu
- [Take a gander at some terminologies and concepts](#take-a-gander-at-some-terminologies-and-concepts)
- [Understand the Unity GetSpectrumData function](#understand-the-unity-getspectrumdata-function)
- [The spectrum datas and the Fourier transform](#the-spectrum-datas-and-the-Fourier-transform)
- [Provide the spectrum datas to the audio visulizer](#provide-the-spectrum-datas-to-the-audio-visulizer)

# Take a gander at some terminologies and concepts

**Sound**: A sort of wave, viberated in the air with a frequency between 20Hz and 20000Hz.  
**Sound Frequency**: The times that sound viberated per second. It's unit is herz.   
**Sampling**: Get a sample, a value, from a sound wave.     
**Sampling Rate**: The times of the Sampling per second.    
**Fast Fourier Transform**: the FFT can used to transform signals.  
**Window Function**: To reduce the "signal to noise ratio".  
**Decibel**: A well-known value used to measure sound's volume. In this artikel, the strength of the sound wave is a relative value and has nothing to do with the decibel.

# Understand the Unity GetSpectrumData function

**public void GetSpectrumData(float[] samples, int channel, FFTWindow window);**  

**samples**: The array is the return value of the funciton. Each element's value is the strength of the audio source at a certain frequency. To be optimized for the Fourier transform algorithm, the length of the array must be the power of two, minimal 64 and maximum 8192.  
**channel**: Stereo seperates a audio source into different parts, or say channels, and provided them to distanced audio hardwares to play. In this case, the channel parameter can be used to choose a certain channel to sample by the FFT. Setting to 0 will sample all channels, as same as in case of mono.  
**window**: A helper function for the FFT to reduce noises. There is a trade-off of the complexity of the window-function for reducing the noises.  
**usage**:
```csharp
public float[] spectrumData=new float[8192];  
thisAudioSource.GetSpectrumData(spectrumData,0,FFTWindow.BlackmanHarris);  
```

# The spectrum datas and the Fourier transform

Given the sampling time T, the samples N and the sampling rate $f_s$, we can make a formula: $T=N/f_s$.  

The reciprocal of T, $f_s/N$, is called Frequency Resolution. The charts below are the results of the FFT of the demo audio. They show that the higher the Frequency Resolution the denser the datas transformed.

![placeholder](https://res.cloudinary.com/dokdkay80/image/upload/v1604498587/AudioVisualizer/av3_vnggql.png)
![placeholder](https://res.cloudinary.com/dokdkay80/image/upload/v1604498510/AudioVisualizer/av4_ljqbsw.png)

The length of the spectrumData array is actually the Frequency Resolution and every array's element has a x/y value in the diagrams above. Unfortunately, we can't get the frequency, the x, of a element(but well the strength which is y) at runtime and I can't find any clue in the Unity documentation either. So let's try to figure it out through some tests.  

Using a software to analyze "[MV] FIESTAR(피에스타) _ Mirror.mp3", we can see that there is a free fall of the strength values at the 16000Hz frequency. 

![placeholder](https://res.cloudinary.com/dokdkay80/image/upload/v1604498406/AudioVisualizer/av5_aze0ep.png)

Back to Unity, Debugging the spectrumdata's values, there is also a significant decrease at spectrumData[5500]. Therefore, it can be deduced that the spectrumData[5500] represences the frequency around 16000Hz. Moreover, 16000Hz/5500\*8192=23831Hz, we can also conclude that the highest frequency the function returned is between 22000Hz to 24000Hz. 

In addtion to the tests, other clues can be found from the angle of the Fourier Transform. The core algorithm of GetSpectrumData function is the FFT. As mentioned in the beginning, sound is a sort of wave where frequency decided pitch and amplitude determined volume. 

![placeholder](https://res.cloudinary.com/dokdkay80/image/upload/v1605801186/AudioVisualizer/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7_2020-11-19_%E4%B8%8B%E5%8D%8811.21.18_k0pcox.png)

The graph above have shown the two sound waves with stable frequency and gradually amplified amplitude in the x-time/y-amplitude plane. According to the superposition principle, simply adding these waves will generate a new complex wave. 

![placeholder](https://res.cloudinary.com/dokdkay80/image/upload/v1605801187/AudioVisualizer/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7_2020-11-19_%E4%B8%8B%E5%8D%8811.22.20_fdt7q0.png)

This is exactly what happend when the different sound waves met in the air and created a mixed tone. The datas of a audio file are just the discrete samples of a complex sound wave. From this perspective, we can say that making audio/music is adding different sound waves into a complex wave. Just the opposite, the Fourier Transform breaks down complex wave. No matter how complex the wave is, through a series of magical calculations(much more complicated than mixing them), the FT can identify every single-frequency wave within it.  

From the viewpoint of two-dimensional space, the Fourier Transform will translate the complex wave from the "x-time/y-amplitude" plane(the last chart above) to the "x-frequency/y-strength"(e.g the third graph) and then to many "x-time/y-strength of a wave with fixed frequency". The strength of every single wave, the y, on the time axis will become the value of the correspond element of spectrumData array.  

In this transfomation process, the FFT firstly needs to sample the original complex wave on the time axis with a fast interval repeatedly. To correctly reconstruct the single-frequency wave, sufficient samples are necessary. Here comes "the Nyquist–Shannon sampling theorem". To put it simply, the theorem states that the number of the sufficient samples is the frequency multiply by two. For instance, for a 3Hz wave you need 6 samples. 

The nowadays music industry adopts the 44100Hz sampling rate standard. So a music audio of one second contains 44100 samples. Because of the restriction of the Nyquist theorem, the highest frequency the FFT can correctly analyze is 22050Hz, which is consistent with the conclusion of the tests above. Given the highest frequency and the length of the spectrumData array, we can further determine the exact frequencies presented by the elements of the array. For example, the demo project has used a 8192 array:  

spectrumData[0]		<=>		22050Hz/8192\*1    = 2.6916 Hz  <=>  6 times sampling per second  
spectrumData[1]		<=>		22050Hz/8192\*2    = 5.3833 Hz  <=>  11 times sampling per second  
...    
spectrumData[8191]	<=>		22050Hz/8192\*8192 = 22050  Hz  <=>  44100 times sampling per second  

The dependecies above also tell us that the longer the array the more the sampling and thus the more performance cost of the GetSpectrumData function.  

Through adjusting the length of the array, any certain frequency's strength can be obtained and hence the analyze can be extended to other dimensions, e.g the pitchs.  

![placeholder](https://res.cloudinary.com/dokdkay80/image/upload/v1605164286/AudioVisualizer/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7_2020-11-12_%E4%B8%8B%E5%8D%882.57.30_vdylrr.png)

According to this <a href="https://en.wikipedia.org/wiki/Scientific_pitch_notation">pitch/frequncy table</a>, the highest pitch's frequncy of octave 9 is 15804Hz, which explained a little about the free fall at 16000Hz of the FFT result's chart above as well.

# Provide the spectrum datas to the audio visulizer

<a href="https://github.com/liu-if-else/UnitySpectrumData">Github Link of the demo project</a>

In the demo project, I obtained the spectrum datas in Update and injected them to the scale of the cubes and the colors of their materials. The workload is distributed to both CPU and GPU. If you want to reduce the burden of CPU, another idea is to translate the whole spectrum datas, the array, to the GPU(since you have to use this bandwidth to set colors anyway) and to hand over all the work to the shaders, like vertex animation, color calculation and etc. Using vertex or fragment shader to calculate the color instead of CPU is also more friendly for batching and will reduce a lot of SetPass calls.  

Since the highest frequncy of the song is below 16000Hz, the audio visualizer only uses the 5461 of 8192 cubes to construct itself.  

The core logics of the demo are in the Controller script:   
<br/>
```csharp
using UnityEngine;
using System.Collections;
using DG.Tweening;

public class Controller : MonoBehaviour {
    //audio related
    public AudioSource thisAudioSource;
    private float[] spectrumData = new float[8192];
    //cube related
	public GameObject cubePrototype;
	public Transform startPoint;
	private Transform[] cube_transforms=new Transform[8192];
    private Vector3[] cubes_position= new Vector3[8192];
    //color related
    public GridOverlay gridOverlay;
    private MeshRenderer[] cube_meshRenderers = new MeshRenderer[8192];
    private bool cubeColorChange;
    private bool gridColorChange;
    //camera movement related
    public Vector3 cameraStartPoint;
    public Transform cameraTransform;
    public bool lookat0_1;
    public bool lookat1_2;
    public bool lookat2_3;
    public Vector3 lookat0_1_vector = Vector3.zero;
    public Vector3 lookat1_2_vector = new Vector3(106f, 12f, 78f);
    public Vector3 lookat2_3_vector = Vector3.zero;
    private Vector3[] moveTos = new Vector3[8192];
    public Transform cubes_parent;
    private bool cubesRotate = true;
	// Use this for initialization
	void Start () {
        //cube generation
		Vector3 p=startPoint.position;

		for(int i=0;i<8192;i++){
			p=new Vector3(p.x+0.11f,p.y,p.z);
            GameObject cube=Object.Instantiate(cubePrototype,p,cubePrototype.transform.rotation)as GameObject;
			cube_transforms[i]=cube.transform;
            cube_meshRenderers[i] =cube.GetComponent<MeshRenderer>();
		}

		p=startPoint.position;

		float a=2f*Mathf.PI/5461;

		for(int i=0;i<5461;i++){
			cube_transforms[i].position=new Vector3(p.x+Mathf.Cos(a)*131,p.y,p.z+131*Mathf.Sin(a));
			a+=2f*Mathf.PI/5461;
            cubes_position[i]=cube_transforms[i].position;
			cube_transforms[i].parent=startPoint;
		}
        //manage color
        gridColorChange = false;
        cubeColorChange = false;
        Invoke("SwitchCC", 3f);
        //manage camera
        cameraStartPoint = cameraTransform.position;
        StartCoroutine(CameraMovement());
        //delay the music
        thisAudioSource.PlayDelayed(2f);
	}
	// Update is called once per frame
	void Update () {
        Spectrum2Cube();
        DynamicColor();
        CameraLookAt();
	}
	//color related
    void SwitchCC(){
        cubeColorChange = !cubeColorChange;
    }
    void SwitchGC(){
        gridColorChange = !gridColorChange;
    }
	void DynamicColor(){
        if (cubeColorChange)
        {
            for (int i = 0; i < 5461; i++)
            {
                cube_meshRenderers[i].material.SetColor("_Color", new Vector4(Mathf.Lerp(cube_meshRenderers[i].material.color.r, spectrumData[i] * 500f, 0.2f), 0.5f, 1f, 1f));
            }
        }
        if (gridColorChange)
        {
            float gridColor = Mathf.Lerp(gridOverlay.mainColor.r, spectrumData[2000] * 1000, 0.5f);
            if (gridColor > 1)
            {
                gridColor = 1;
            }
            gridOverlay.mainColor = new Vector4(gridColor, 0.5f, 1f, 1f);
        }
    }
    //inject spectrumDatas to the scale of the cubes
    void Spectrum2Cube(){
        thisAudioSource.GetSpectrumData(spectrumData, 0, FFTWindow.BlackmanHarris);
        for (int i = 0; i < 5461; i++)
        {
            cube_transforms[i].localScale = new Vector3(0.15f, Mathf.Lerp(cube_transforms[i].localScale.y, spectrumData[i] * 10000f, 0.5f), 0.15f);
        }
    }
    //manage the angles of the camera
    void CameraLookAt(){
        if (lookat0_1)
        {
            cameraTransform.LookAt(lookat0_1_vector);
        }
        if (lookat1_2)
        {
            cameraTransform.LookAt(lookat1_2_vector);

        }
        if (lookat2_3)
        {
            cameraTransform.LookAt(cubes_position[5190]);
        }
    }
    //animate the grid
    IEnumerator GridOff()
    {
        for (int i = 0; i < 51; i++)
        {
            gridOverlay.largeStep += 10;
            yield return new WaitForSeconds(0.02f);
        }
        gridOverlay.showMain = false;

    }
    IEnumerator GridOn()
    {
        gridOverlay.showMain = true;
        gridColorChange = true;
        gridOverlay.largeStep = 500;
        for (int i = 0; i < 49; i++)
        {
            gridOverlay.largeStep -= 10;
            yield return new WaitForSeconds(0.02f);
        }
    }
    //repeat the camera movement. TODO: exit mechanism
    public void CameraRepeatMove()
    {
        StopAllCoroutines();
        StartCoroutine(CameraMovement());
        if (cubesRotate)
        {
            cubesRotate = false;
            cubes_parent.DORotate(new Vector3(0f, 360f, 0f), 117f, RotateMode.FastBeyond360);
        }
        gridColorChange = false;
    }
    //move the camera
    IEnumerator CameraMovement()
    {
        yield return new WaitForSeconds(20f);
        lookat2_3_vector = new Vector3(cubes_position[5200].x, 12f, cubes_position[5200].z);
        cameraTransform.DOMove(startPoint.position, 20f);
        for (int i = 0; i < 8192; i++)
        {
            moveTos[i] = new Vector3(cubes_position[i].x, 10f, cubes_position[i].z);
        }
        yield return new WaitForSeconds(20f);
        cameraTransform.DOMove(new Vector3(126f, 252f, 1f), 10f);
        cameraTransform.DOLookAt(Vector3.zero, 10f, AxisConstraint.None, Vector3.up);
        yield return new WaitForSeconds(10f);
        cameraTransform.DOMove(new Vector3(106f, 12f, 78f), 19f);
        cameraTransform.DOLookAt(lookat1_2_vector, 19f, AxisConstraint.None, Vector3.up);
        yield return new WaitForSeconds(19f);
        lookat1_2 = false;
        StartCoroutine(GridOn());
        cameraTransform.DOLookAt(lookat2_3_vector, 8f, AxisConstraint.None, Vector3.up);
        cameraTransform.DOMove(new Vector3(cubes_position[5460].x, 12f, cubes_position[5460].z), 8f);
        yield return new WaitForSeconds(8f);
        cameraTransform.DOLookAt(cubes_position[5200], 2f, AxisConstraint.None, Vector3.up);
        yield return new WaitForSeconds(2f);
        int counter = 0;
        while (counter < 2700)
        {
            cameraTransform.LookAt(cubes_position[5200 - counter]);
            cameraTransform.DOMove(moveTos[5460 - counter], 0.01f);
            yield return new WaitForSeconds(0.01f);
            counter += 10;
        }
        cameraTransform.DOLookAt(lookat0_1_vector, 3f, AxisConstraint.None, Vector3.up);
        yield return new WaitForSeconds(3f);
        StartCoroutine(GridOff());
        lookat0_1 = true;
        cameraTransform.DOMove(new Vector3(cameraStartPoint.x, cameraStartPoint.y + 300f, cameraStartPoint.z), 6f);
        yield return new WaitForSeconds(6f);
        lookat0_1 = false;
        CameraRepeatMove();
    }
} 
```

Screenshots from the Editor:

![placeholder](https://res.cloudinary.com/dokdkay80/image/upload/v1604498463/AudioVisualizer/av2_r15mar.png)

![placeholder](https://res.cloudinary.com/dokdkay80/image/upload/v1604498366/AudioVisualizer/av1_itqijr.png)

----
**References:**  
the Nyquist–Shannon sampling theorem — Zero to One  
<a href="https://www.cnblogs.com/zoneofmine/p/10853096.html">https://www.cnblogs.com/zoneofmine/p/10853096.html</a>  

Algorithmic Beat Mapping in Unity: Real-time Audio Analysis Using the Unity API — Jesse  
<a href="https://medium.com/giant-scam/algorithmic-beat-mapping-in-unity-real-time-audio-analysis-using-the-unity-api-6e9595823ce4">https://medium.com/giant-scam/algorithmic-beat-mapping-in-unity-real-time-audio-analysis-using-the-unity-api-6e9595823ce4</a>  

What is the Fourier Transform — 3Blue1Brown  
<a href="https://www.youtube.com/watch?v=spUNpyF58BY">https://www.youtube.com/watch?v=spUNpyF58BY</a>  

----
**Chinese Version of this artikel:**  
<a href="https://blog.csdn.net/liu_if_else/article/details/51233799">https://blog.csdn.net/liu_if_else/article/details/51233799</a>  
