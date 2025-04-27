# Sauer-Vslot-Text-Sender
Store and share text files of unlimited sizes using [Sauerbraten](http://sauerbraten.org/)'s map geometry.

**Vslot Text Sender (VSTS)** is a set of commands that allows you to encode text into the geometry of Sauerbraten maps, enabling you to share them with `/sendmap` and retrieve them with `/getmap`.

It is common practice in Sauerbraten to use the `/do $maptitle` command to share small scripts within the game, but one limitation of `maptitle` is its character limit. Any content that exceeds 259 characters will be truncated when attempting to save or send the map.

**VSTS** solves this problem by using the Vslot system to store all the content.

More specifically, it makes use of the `vshaderparam` and `getvshaderparamnames` commands. You can see a more detailed explanation [here](#vsts-inner-workings).

# How to Install
- Download [vsts.zip](https://github.com/CubeScript/Sauer-Vslot-Text-Sender/releases/latest/download/vsts.zip) and move it to the root folder of your Sauerbraten.
- Add the command `addzip "vsts.zip"; exec vsts.cfg` to your `autoexec.cfg` (you can open it with `/notepad autoexec.cfg`, edit, save, and execute it.)
- Once the installation is done, there will be a map that can be loaded with `/coop vsts_shaders` and an additional menu that can be opened with `/vshadereditor`.

<details>
<summary>Other optional ways to install VSTS</summary>

- Optionally, you can unzip the file and just execute `vsts.cfg` however you prefer.

- Or even (and this is totally unnecessary) you can load the `vsts_shaders` map and run `/do $maptitle`, since all the contents of `vsts.cfg` are included in the geometry :)

- If you don't feel like playing around with shaders, you can [install the vsts_lite.cfg file](https://github.com/CubeScript/Sauer-Vslot-Text-Sender/releases/latest/download/vsts_lite.cfg), which only has the essential **VSTS** commands — no GUI, no shaders.

</details>

<hr>

# VSTS commands

- ### `/vsts_write [content] [chunk size]`

    This is the main command to write and store text content in the map. You must be in edit mode and have some geometry selected.

    `content` can be a text of any size, and **VSTS** will ensure it is stored in its entirety.

    `chunk size` is an optional integer that determines how much text will be stored in each `vshaderparam` command. By default, it's 250 (more info [here](#encoding)). There's no need to change this value unless you're experimenting.

    After applying the content to a geometry, you must get the slot ID where it is stored by using `/echo (getseltex)` on the same geometry where you applied it. You will use this ID later to retrieve the content.

    Example:
    ```cs
    /vsts_write "Hello, this is a short message, it could easily be longer. In fact, there is room for well over 259 characters here. It's nice to have the space, but I don't have much to say right now. Try it out for yourself and see what you can come up with... This message is 283 characters long!"
    ```
    Tip:
    check out [vsts_read](#vsts_read-slot-id-or-string) below to see how to read the value back.

- ### `/vsts_get_by_slot [slot ID]`
    This command returns all the encoded content stored in a slot using its ID.

    Example:
    ```cs
    // assuming we have a VSTS stored in slot 1704
    /echo (vsts_get_by_slot 1704)
    ```

- ### `/vsts_get_by_match [string] [start] [end]`
    This command allows you to find a slot based on the stored text content without needing to specify an ID. It returns all the encoded content. You can optionally specify a `start` and `end` to limit the search range within the content.

    Example:
    ```cs
    /echo (vsts_get_by_match "a short message")
    ```

- ### `/vsts_code_to_string [encoded string]`
    This is the command you can use to decode the content after retrieving it with `vsts_get_by_slot` or `vsts_get_by_match`.

    Example:
    ```cs
    /echo (vsts_code_to_string (vsts_get_by_match "a short message"))
    ```

- ### `/vsts_string_to_code [plain text]`
    This is the command used by `vsts_write` to encode the text before storing it.

- ### `/vsts_read [slot ID or string]`
    It uses all the previous commands to simplify reading the content already written in the map. You only need to specify a slot ID or a piece of text that matches some content. The content will be decoded and returned as the original text.
    Returns a list if multiple matches are found.

    Example:
    ```cs
    /echo (vsts_read "a short message")
    // or:
    /echo (vsts_read 1704)
    ```


<hr>

# VSTS inner workings

**VSTS** mainly relies on two commands: `vshaderparam` and `getvshaderparamnames`, which, although typically used to define and retrieve shader parameters from a texture, are used here to store any kind of text content.

- `vshaderparam` takes five arguments. The first is a name, then a sequence of four float values that act as the uniform parameters.

    ```
    /vshaderparam "hello" 0.0 512.0 0.0 0.0
    ```

- After applying this to some geometry, we can retrieve those values using the `getvshaderparam` and `getvshaderparamnames` commands.

    - `getvshaderparam` needs two arguments: the vslot index (which we can grab with `getseltex`), and the name of the uniform:

        ```
        /echo (getvshaderparam (getseltex) "hello")
        ```

        Result: `0.0 512.0 0.0 0.0`

    - `getvshaderparamnames` takes a vslot index and gives us back all uniform names in that slot:

        ```
        /echo (getvshaderparamnames (getseltex))
        ```

        Result: `hello`

The way **VSTS** works is by using the *name* parameter, which—just like `maptitle`—only supports 259 characters. However, since we can add multiple `vshaderparam` commands to the same slot (using `vdelta [vshaderparam ... ; vshaderparam ...]`), we can split the text into 259-character "chunks" and concatenate them later.

### Encoding

To write/encode text into a slot, **VSTS** uses the following format:

Let’s suppose we have the string "cube 2 sauerbraten". `vsts_string_to_code` will iterate over each character in this string and convert it to its numeric representation using `strcode`. If the numeric representation has fewer than 3 digits, a `!` character is added to the right of the number to keep every numeric segment in 3-character chunks. This makes decoding easier.


So as a result of encoding "cube 2 sauerbraten", we will get:
```
99!11798!10132!50!32!11597!11710111498!11497!116101110
```

`vshaderparam` could easily support raw text without the need for conversion, but special characters aren’t handled very well, so this is the best way to store and retrieve content without extra headaches.

The numeric string will be stored without issues within a 259-character chunk. However, if for some reason you have content with more than 259 characters where one or more of its chunks are repeated, you’ll run into problems because `vshaderparam` only supports one identical name per slot.

To solve this, when executing `vsts_write`, **VSTS** appends random amounts of an extra `-` character to ensure that each `vshaderparam` has a unique name value.

`vsts_write` also adds the character `[` at the beginning of the name and the character `]` at the end, to make it easier to read multiple `vshaderparam`s in the same slot without mixing them up. This effectively encapsulates all the chunks related to the same piece of text.

If we assume that the chunk size is 10 characters (it isn’t, but let’s assume that just to make visualization easier), the encoded content would look like:

```
[9-9--!-117 98!1----0- 13--2-!-5- 0--!32-!11 5--9--7--! -11-71-0-- 1-11-49-8! 11--497!11 ----6-1011 -1-0-]
```
This is what the map stores, and it’s what you’ll see if you try to read the `getvshaderparamnames` of the slot.

### Decoding

Each whitespace represents a chunk (that is, a new `vshaderparam` command executed), and the `-` characters were inserted randomly to help keep the names unique.

When decoding these values, we use `strreplace` to remove the spaces and `-` characters, keeping only the numbers and the `!` signs. Once that’s done, we’re free to decode the numeric string.

After the `strreplace` we get back to:
```
[99!11798!10132!50!32!11597!11710111498!11497!116101110]
```
(if there are multiple texts, they will be returned in their respective [blocks])

We know that each 3-digit numeric segment represents one original character, so to decode we use `vsts_code_to_string`, which iterates through the numeric string every 3 characters and converts it back to the original string using `codestr`, which automatically ignores the additional `!` character.

```
99! -> c  
117 -> u  
98! -> b  
101 -> e  
32! -> (space)
50! -> 2  
32! -> (space)
115 -> s  
97! -> a  
117 -> u  
101 -> e  
114 -> r  
98! -> b  
114 -> r  
97! -> a  
116 -> t  
101 -> e  
110 -> n
```
And that’s all we need to do to retrieve the original text: "cube 2 sauerbraten".

### Important Considerations
- To keep slot synchronization between players after the `compactvslot` command, you may sometimes need to run `/sendmap` if you notice something unusual. The best practice is to ask the player to select the geometry where you stored your content, so they can see the slot ID (`getseltex`) assigned to that geometry in their context, which may be different from yours if they haven’t executed `/getmap`. Sometimes a `/compactvslots` is enough to synchronize.

- If `/sendmap` or `/savemap` takes forever, it means someone wrote a huge amount of content into some geometry and you didn’t run `/compactvslots` (it’s automatically executed when you use `/vsts_write`). In short, `compactvslots` is your best friend for keeping things in order.

- **VSTS** won’t execute anything without your action, don’t worry. But it’s entirely your responsibility to check the stored content if you intend to execute it with `/do`.

- It's all experimental, expect something to go wrong at some point :)

<hr>

# VSTS Shader Editor

There is a menu available as a proof of concept to enable sharing GLSL code between players using the **VSTS** system. It allows you to control which shaders from other players should be applied, as well as create your own or edit existing ones.

GLSL allows you to create all kinds of effects on textures, when you create a new shader slot, you will have this text by default:

```glsl
//@id 5

//@fragment
color = vec4(0, 0, 1, 0);

//@vertex
position.z += 16;
```

It’s a type of markup that allows you to define which geometry the shader should be applied to using an ID (in this case, 5 by default), the Fragment Shader, and the Vertex Shader (it doesn’t matter if you don’t know what these are for now, just keep in mind that things related to position come after `//@vertex`, and things related to color come after `//@fragment`).

Through the menu, you can define the ID of the selected geometry, the position, and the radius of the trigger’s range.

**The effect will only work on textures that have the shader enabled (with setshader), you can find them on the last page of the F2 menu.**

If the trigger is enabled, the default example above will color the geometry blue and will add +16 units of distance on the Z axis of its position.


Another example (wave animation):
```glsl
//@id 5

//@vertex
position.y += sin(vvertex.x * (millis * 0.01)) + 2;
```
This GLSL relies on the handy uniform `millis` which returns a time value, in conjunction with the `vvertex` which holds the current vertex position, we can make a wave animation using the math function `sin()`.

There are several useful data sources like:
| **Name**              | **Scope**       | **Description**                                                                 |
|-----------------------|-----------------|---------------------------------------------------------------------------------|
| `diffuse`             | fragment        | Fragment color sampled from `diffusemap` using `texcoord0`.                    |
| `maplight`            | fragment        | Fragment color sampled from `lightmap` using `texcoord1`.                      |
| `color`               | fragment        | Same as `gl_FragColor`, defines the final fragment color.                      |
| `colorparams`         | fragment        | Color value set by `vcolor` command.                                           |
| `position`            | vertex          | Final vertex position offset (calculated in vertex shader).                   |
| `vvertex`             | vertex          | Original position of the vertex.                                               |
| `geometryposition`    | vertex          | Custom uniform (set via `/vdelta [vshaderparam geometryposition x y z rotation_type]`) that offsets geometry.         |
| `geometrypivot`       | vertex          | Custom uniform (set via `/vdelta [vshaderparam geometrypivot x y z rotate]`) that defines rotation pivot.   |
| `trigger`             | vertex          | Custom uniform (set via `/vdelta [vshaderparam trigger x y z id]`) for the trigger's center.      |
| `triggerradius`       | vertex          | Custom uniform (set via `/vdelta [vshaderparam triggerradius width depth height]`) for the trigger's radius.      |
| `camera`              | fragment/vertex | Holds camera position.                                                         |
| `millis`              | fragment/vertex | Time in milliseconds since the start of the game.                              |
| `dist`                | fragment/vertex | Distance between the camera and the vertex (`is_triggered.y`).                |
| `is_active`           | fragment/vertex | `true` if the camera is inside the trigger box (`is_triggered.x == 1`).       |
| `normal`              | fragment/vertex | Surface normal passed from vertex to fragment shader.                          |
| `camvec`              | fragment/vertex | Vector from vertex to camera (`camera - vvertex`).                             |
| `camdir`              | fragment/vertex | Normalized direction from vertex to camera (`normalize(camvec)`).             |
| `camprojmatrix`       | fragment/vertex | Camera projection matrix for converting to screen space.                       |
| `texcoord0`           | fragment/vertex | Texture coordinates for the base texture (`diffusemap`).                       |
| `texcoord1`           | fragment/vertex | Texture coordinates for the lightmap.                                          |

Tip: if you move a vertex and it should be visible, but it is still hidden (occluded), it is because the original geometry is off-screen. `/oqgeom 0` disables the occlusion system for the geometry.

The menu workflow works as follows:

- After installing **VSTS**, type `/vshadereditor` to open the menu;
- Click the `[+]` button to create a **VSTS** slot;
- Set a description and a filename;

    Shader files must have the `.glsl` extension, and files ending with `.cfg` will display a button to execute them.

- Customize the body of your VSTS with any content you wish to share;
- Optionally save the file;

    Saving it will create an individual file in the `vsts` folder inside your Sauerbraten home. If you don't save it, the content will be saved into `config.cfg` when allowed, and it may be accidentally deleted later.

- Click `[allow]` so that (in the case of shaders) the content takes effect locally;
- Select another geometry and click `[share from local save]` so that the content is effectively written into the map for other players to access;

    After a few seconds, your **VSTS** should appear in the main menu for all players. If it doesn't happen, type `/sendmap` and ask other players to run `/getmap`.

Common confusions:

- Shader effects only work on textures that have a shader applied (using `setshader`). You can see them on the last page of the texture browser (press F2 in edit mode);

    Make sure you apply the texture <u>before</u> setting the triggers, otherwise you will lose them.

- `trigger pos` and `trigger size` settings are individual for each geometry and are not shared with the **VSTS**, nor even stored when saving. They will be lost if you modify the texture of the geometry. Console messages will indicate if something is missing, but pay attention to the color indicators:

    - Gray text & colored numbers: you are not in edit mode;
    - Gray text & gray numbers: you are in edit mode, but not selecting any geometry;
    - Orange text & gray numbers: you are in edit mode and are selecting a geometry that does not yet have any trigger applied;
    - Orange text & orange numbers: you are in edit mode and are selecting a geometry that already has some trigger configuration applied;

- Sometimes it may not be possible to write the **VSTS** into the same geometry where you are configuring `trigger pos` and `trigger size`. In such cases, you should select a different geometry to act as the "host";

- A POSTFX wireframe effect is shown to indicate the trigger’s position and size. You can clear it by opening the menu with `/vshadereditor` outside of edit mode, or by running `/clearpostfx`;

Just as another proof of concept, there is a map called `vsts_shaders.ogz` included that contains all of the **VSTS** encoded into the geometry and some cool effects, there is a decoder in the maptitle that you can run with `/do $maptitle` to get the menu.
maptitle is used in this case because the decoder is small and is not affected by any of the limitations mentioned earlier.

However, it is recommended to use the vsts.cfg from this repository, which is up to date and does not require you to /do $maptitle.

and that's all!