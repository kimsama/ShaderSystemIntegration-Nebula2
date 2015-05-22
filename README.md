
A backup of the article which was contributed on [ShaderX5](http://www.shaderx5.com/), published in 2006.


# Shader System Integration: Nebula2 and 3ds Max

*Kim Hyoun Woo*

Introduction
------------
This article presents ideas for integrating shader systems into DCC tools for artist control. As a case study, we take a look over the art pipeline workflow established by nmaxtoolbox, the Nebula2 plug-in for 3ds Max. The Nebula Device [Nebula06] is an open-source real-time 3D game/visualization engine, written in C++. Version 2 has a modern rendering engine that makes full use of shaders. Due to its shader-centric nature, it is also important to provide a shader-driven system for the 3D art pipeline.

The system described here should be considered by both programmers and artists, as it has the following key features:

* It proposes an easy and flexible way to integrate a shader system into any 3D art package.
* It should take a little time to implement, and should easily integrate into existing code bases.
* It does not require modifications to existing shader code (e.g. for exposing UI controls in the 3D art package for shader parameters).
* It is independent of the actual shader code. Shaders can be written in any language and still get exposed properly for preview and tweaking.
* It shows how to provide an easy to use UI for specifying shader parameters.
* It provides users with instant feedback as they tweak shader parameters from within the 3D art package.
* It does not require the use of DCC tools SDKs to do the integration.

Art Pipeline Integration
------------------------
Since artists are the ones responsible for the final quality of art assets, then it is important that a 3D art package allows them to tweak shader parameters in order to obtain the desired look for what they create. For this reason, most DCC tools provide facilities for implementing such systems.
For example, with 3ds Max 6 (or later versions), it is possible to expose HLSL shader parameters from effect files. The effect file is loaded and analyzed with accordance to DXSAS [DXSAS01] to get shader parameters exposed for artist control. This is a powerful feature. However, it is limited to DXSAS-compliant shaders only.

In the case of nmaxtoolbox, we use a different workflow as shown in figure 1 below.

[INSERT FIGURE 1 HERE]
Figure 1. Workflow of the Nebula2 3ds Max Toolkit.

The diagram above shows that effect files (which are written by a shader programmer) are copied to a predetermined location that the game engine knows about. The plug-in inside the 3D art package then dynamically generates UI controls for tweaking shader parameters of a particular shader (in our case, this plug-in is nmaxtoolbox). The UI information is extracted by parsing a shaders.xml file (which is created by a programmer). Finally, the game engine depends on the renderpath.xml file to know where it should look for the actual shader code.

To suffice artists’ needs, the plug-in must support at least two things:
* Easy-to-use user interface for tweaking shader parameters.
* Real-time feedback during shader parameters tweaking.

In the case of 3ds Max, perhaps one of the easiest ways to provide a decent user-interface for editing shader parameters is to use the program’s scripting facility.
For nmaxtoolbox, the parameters for each shader are described in an XML meta file (shaders.xml). This meta file contains the name, type, and the default value for each parameter, as well as additional information that is useful for the plug-in.
At load time, the plug-in then automatically generates MAXScript code based on the given meta file’s contents.

A typical meta file would look like the following:

```xml
    <shader name="Standard" shaderType="default" meshType="default bsp" file="static">
        ...
        <param name="MatDiffuse" label="Diffuse Color" type="Color" gui="1" export="1" def="1.0 1.0 1.0 1.0" />
        <param name="MatEmissive" label="Emissive Color" type="Color" gui="1" export="1" def="0.0 0.0 0.0 0.0" />
        <param name="MatEmissiveIntensity" label="Emissive Intensity" type="Float" gui="1" export="1" min="0.0" max="10.0" def="1.0" />
        <param name="MatSpecular" label="Specular Color" type="Color" gui="1" export="1" def="0.5 0.5 0.5 1.0" />
        ...
    </shader>
```

For the “Standard” shader, the parameter description for “MatDiffuse” contains the name attribute (which specifies the name of the parameter), the label attribute (which specifies the label to use for the corresponding UI control), the type attribute (which specifies the data type of the parameter) where Color means that a color picker should be used for changing the value of the parameter. Finally, the def attribute specifies the initial/default value of the parameter. Figure 2 below shows the user-interface that is generated for this shader inside 3ds Max.

[INSERT FIGURE 2 HERE]
Figure 2. Nebula2’s UI controls inside 3ds Max’s Material Editor.

The good thing about this design is that it frees the UI from the underlying shader language, thus the shader can be freely written in any language (e.g. HLSL or CgFX). Also, the same UI meta files can be used across different 3D art packages…

UI controls are added by means of MAXScript. However, it can be tedious to write new MAXScript code every time a new shader is to be exposed. Thus, the plug-in contains logic to automatically generate MAXScript code for UI controls out of meta files.
The controls created by the plug-in are of different types that suit the underlying data type. These controls include spin buttons, drop-down lists, texture buttons, color pickers, and static labels.

Most 3D art packages have a scripting engine which can be used to dynamically create the UI controls. However, in the case where scripting is not available, usually plain C++ code generation can compensate for that but with some more added effort.

[INSERT FIGURE 3 HERE]
Figure 3. Nebula2 Maya Material Editor.

[INSERT FIGURE 4 HERE]
Figure 4. Nebula2 LightWave Material Editor.

For Maya, the Nebula2 exporter plug-in uses MEL (Maya’s scripting language) to generate UI controls for shader parameters [NebulaMayaTookit06]. However, the LightWave plug-in just uses plain C++ code to achieve the same task.
So in the end, to expose a new shader, we only have to add a new shader entry to the meta file, and then to specify the shader parameters. The required UI script/code will be automatically generated then. So adding new shaders can be done even by an artist.

As an example, the code snippet below shows the generated MAXScript code for the Standard material:

```
    caStandard = attributes "Standard"
    (
    parameters Standard rollout:rStandard
    (
        ...
        MatDiffuse type:#frgba default:[255.000000, 255.000000, 255.000000, 255.000000] ui:MatDiffuse 
        on MatDiffuse set val do 
        (
            if loading != true do
            (
                curMaterial = medit.GetCurMtl()
                if classof curMaterial != MultiMaterial do
                    if curMaterial.delegate != undefined do
                       curMaterial.delegate.ambient = val
            )
        )
        ...

        rollout rStandard "Standard Parameters"
        (
            ...
            colorpicker MatDiffuse "Diffuse Color" align:#left alpha:true color:[255.000000, 255.000000, 255.000000, 255.000000] 
            colorpicker MatEmissive "Emissive Color" align:#left alpha:true color:[0.000000, 0.000000, 0.000000, 0.000000]
            ...
        )
    )
```

Data Exchange
-------------
All exposed shader parameters should be correctly serialized when the 3D model is exported. Thus, the plug-in must be able to access and store shader parameters that are associated with a particular art asset. For 3ds Max, this can be accomplished with custom attributes, while for Maya it is extra attributes. These are additional pieces of information that are saved along with the relevant 3D model in the scene file.

Note that the plug-in exports only the alias of the shader, not the shader code itself. The mapping of shader aliases-to-effect files is specified in the XML meta file (renderpath.xml) which is used by the rendering engine. A typical renderpath.xml meta file might look like this:

```xml
    <RenderPath name="dx9hdr" shaderPath="home:data/shaders/2.0">
        ...
        <!-- declare shaders and technique aliases -->
        <Shader name="passes" file="shaders:passes.fx" />
        <Shader name="phases" file="shaders:phases.fx" />
        <Shader name="compose" file="shaders:hdr.fx" />
        <Shader name="static" file="shaders:shaders.fx" />
        <Shader name="static_atest" file="shaders:shaders.fx" />
        <Shader name="environment" file="shaders:shaders.fx" />
        <Shader name="lightmapped" file="shaders:shaders.fx" />
        <Shader name="lightmapped2" file="shaders:shaders.fx" />
        <Shader name="radiosity_normalmapped" file="shaders:shaders.fx" />
        <Shader name="skinned" file="shaders:shaders.fx" />
        <Shader name="blended" file="shaders:shaders.fx" />
        <Shader name="alpha" file="shaders:shaders.fx" />
        ...
    </RenderPath>
```
So for example, if an exported model specifies ‘static’ as its shader alias, then it will end up using the effect file shaders.fx.

The following Nebula TCL script creates a typical Nebula 2 model:

```tcl
    new nshapenode model_0
    sel model_0
    .settexture "DiffMap0" "textures:materials/checker.dds"
    .settexture "BumpMap0" "textures:materials/bump.dds"
    .setvector "MatDiffuse" 1.000000 1.000000 1.000000 1.000000
    .setvector "MatEmissive" 0.000000 0.000000 0.000000 0.000000
    .setfloat "MatEmissiveIntensity" 1.000000
    .setvector "MatSpecular" 1.000000 1.000000 1.000000 1.000000
    .setfloat "MatSpecularPower" 18.160000
    .setint "CullMode" 2
    .setfloat "BumpScale" 0.000000
    .setshader "static"
    .setgroupindex 0
    sel ..
```

The script above shows which texture file to use, as well as the value for diffuse, emissive, specular color…etc. Most of these settings are passed as parameters to the shader, which is also specified in the script. So, model_0 above will be rendered with the static shader (found in shaders.fx as specified by renderpath.xml).

Real-time Feedback
------------------
	Instant feedback for shader parameters tweaks is essential for artists’ productivity. The way to achieve this is to use the game-engine to display the art asset while the artist is working on it, with shader parameter changes being directly reflected in the view as they occur [Bahnassi05]. However, such an implementation requires the use of the 3D art package’s SDK, which can be time-consuming in some cases.
In the case of nmaxtoolbox, the asset in question is exported to a predetermined location on disk, from where a preview window picks it up and loads it for rendering using the game engine.
In comparison with 3ds Max’s hardware plug-in implementation (e.g. the DirectX9 Material), this method does not provide instant feedback for topology modifications (the model has to be re-exported again). However, both implementations provide instant feedback for shader parameter value changes that are done inside of 3ds Max’s Material Library. Since nmaxtoolbox uses 3ds Max parameter blocks to represent shader parameters, then all of these parameters are automatically saved inside the .max file when the scene is saved, which preserves their values across different scene sessions.

[INSERT FIGURE 5 HERE]
Figure 5. The Nebula2 Preview Window rendering the exported model (The opelblitz model is used under permission of RadonLabs).

In figure 5 above, artists can use 3ds Max’s color picker to change the diffuse color of the model. These changes are immediately sent to the preview window for instant feedback on the new model's appearance.
Since the preview window is implemented as an external process, nmaxtoolbox uses Inter-Process Communication (or IPC) to communicate shader parameter changes to the preview window. So, after an art asset gets exported, the plug-in opens the preview window and connects to it using IPC before it starts to send shader parameter change messages. The underlying mechanism of this feature is based on networking technology (TCP). In 3ds Max, parameter changes trigger a MAXScript that generates a Nebula2 script which is then sent to the preview window. Below is an example of such MAXScript code:

```
    if nIsConnectedIpc() do
    (
        nChangeShaderParameter "Standard" "common" "MatDiffuse" "0.5 1.0 1.0 1.0"
    )
```

The script function nChangeShaderParameter is a 3ds Max script-callable function. It wraps the Nebula2 API and sends Nebula2-script with the given parameters to the connected preview window. In the code above, the first parameter “Standard” is the shader type, while the third one “MatDiffuse” specifies which shader parameter to change, and the last parameter is the actual new value for diffuse.

Implementation Notes
--------------------
The source code for both the Nebula2 engine and the nmaxtoolbox plug-in is available free-of-charge for both commercial and non-commercial use on [Nebula06].

So far, this article only described the integration with 3ds Max. But the approach here can be easily adopted for other 3D art packages as well (e.g. Maya, Softimage, Lightwave …etc). In fact, nmaxtoolbox uses the same UI meta file that is used by the Nebula 2 Toolkit for Maya (a commercial Maya plug-in from Radon Labs) and the Nebula 2 Exporter for Lightwave. It is worth mentioning that this method should integrate easily into existing toolchains, including those that use third-party middleware.

One problem with the method described here is that it is hard to have the preview window view and tweak a complex scene containing multiple shaders.
Another potential improvement is to use a third-party script language such as Python or Lua for generating MAXScript out of UI meta files. Currently, whenever the nmaxtoolbox’s UI generation code is modified, it needs to be recompiled and 3ds Max must be restarted. Using a third-party scripting language would allow modifications to the shader UI generation code without recompiling the plug-in or restarting 3ds Max.

Conclusion
----------
This article proposed a system design for integrating game engine visualization services inside DCC tools for artistic shader control and tweaking. The implementation of such a system requires only a small amount of coding. The case of Nebula2 and 3ds Max was taken as an example.
This system is not restricted to a specific shader language. Available shaders are specified in a special file, which is loaded by the nmaxtoolbox plug-in inside 3ds Max. The plug-in then reads meta UI files to render UI controls inside 3ds Max’s Material Library by automatic generation of MAXScript instructions.
A preview window is launched after the 3D object gets exported in order to load and render that object using the game engine. This window is connected with the nmaxtoolbox plug-in to communicate shader parameter changes and give instant feedback as soon as they occur.

Acknowledgements
----------------
I would like to thank RadonLabs for their efforts on The Nebula Device open-source 3D engine. Also, thanks to Vadim Macagon (the author of The Nebula2 LigihtWave Toolkit) for his help with this article.

References
----------
[Nebula06] The Nebula Device, available online at http://www.nebuladevice.org.
[DXSAS01] “DirectX Standard Annotations and Semantics Reference”, DirectX 9 SDK Documentation, Microsoft Corp., 2006.
[NebulaMayaTookit06] Nebula2 Toolkit 2.0 for Maya®, available online at http://www.radonlabs.de/toolkit.html.
[Bahnassi05] Homam Bahnassi and Wessam Bahnassi, “Shader Visualization Systems for the Art Pipeline”, ShaderX3: Advanced Rendering with DirectX and OpenGL, Wolfgang Engel, ed., Charles River Media, 2005, pp. 487-504.
