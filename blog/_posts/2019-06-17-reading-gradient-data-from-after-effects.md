---
title:  "Reading Gradient Data from After Effects"
author: mathieu_mallet
---

Support for shape layers was recently added to You.i Engine. This was done, however, without complete support for all features. This post is about how support for Gradients was added.

![Simple Gradients in AE](../../../assets/img/gradients-in-after-effects/simple_gradients.png)

* TOC
{:toc}

# Starting Up

Adding support for gradients seemed simple enough. There was already code present to read data from other shape layer types, and the vector renderer used by You.i Engine ([NanoVG](https://github.com/memononen/nanovg)) already supported gradient fills and strokes. Implementation was started by adding a new shape type to the Protobuf schema, and then adding new code to the exporter to read the gradient shape data. This is where a major problem was found: while After Effects provides access to most of the gradient data, such as the gradient type and positioning information, there is no Javascript API for accessing the gradient colors.

Digging deeper, this seemed like a problem that other people had [complained about](https://forums.adobe.com/thread/1438128). In short, any property that gets configured through an external dialog (such as gradient colors and line dashes) just cannot be read through the After Effects API. Neither the Javascript nor the C++ APIs support this.

After a lot of googling, I eventually found out that the [bodymovin](https://aescripts.com/bodymovin/) plugin somehow _does_ get access to the data. How does it do it? It was time to dig into the bodymovin source code.

# Bodymovin

Bodymovin in a plugin by Air BnB that allows a user to export After Effects animations as JSON. And somehow they were able to access gradient color data! How did they do it? Luckily the [source](https://github.com/chenqingspring/bodymovin) of the plugin is available. Digging into that the file [ProjectParser.jsx](https://github.com/airbnb/lottie-web/blob/d192bc06d1544c088af073c230a6acf0d6a83303/extension/jsx/utils/ProjectParser.jsx) yielded this snippet:

```javascript
    function getProjectData(){
        var proj = app.project;
        var ff = proj.file;
        var demoFile = new File(ff.absoluteURI);
        demoFile.open('r', 'TEXT', '????');
        fileString = demoFile.read(demoFile.length);
    }


    function getGradientData(shapeNavigation, numKeys){
        if(!fileString){
            getProjectData();
        }
        ...
    }
```

I knew that After Effects projects can be saved as XML files, so this must be a way to access that! Saving a `.aep` file that contains gradient shapes as XML yielded the jackpot: gradient color data gets exported as HTML-encoded XML within the XML file itself.

# Getting the XML

Now the trick was to access the XML data. A quick search of the APIs revealed no way to save a project as XML programmatically. However, the bodymovin plugin was doing it, so it must work right? The Javascript `open` function must have been intelligently returning XML data instead of binary. With that I started writing code again. Since I didn't want to be parsing XML in Javascript, I was going to read the file in Javascript and then transfer it into C++ to parse it with TinyXML2.

This didn't really work out.

When trying to transfer the Javascript string to C++, I ran into multiple problems. It appeared that the string contained some binary data, which might have been due to Adobe including some sort of binary header or footer with the XML data. The solution to this was to encode the string with Base64 before transmitting it to C++. However upon decoding I found something disappointing: the data I received wasn't for an XML export of the After Effects project, it was for the binary data of the `.aep` file. Doh.

# What Now

At this point I dug deeper into what bodymovin was doing. As it turns out, the were reading binary data from the `.aep` file and then doing a text search for the name of the gradient shape. Since those names appear verbatim in the `.aep` file, they could use those to locate the gradient color data XML (which also shows up in plain text in the binary file for some reason). This approach has a few problems:

- Gradient shapes with identical names would cause issues
- Shape layers with identical names couldn't be distinguished from each other
- Compositions with identical names couldn't be distinguished from each other
- Compositions or layers named with the Gradient Fill or Gradient Stroke names would cause issues

This would also be somewhat fragile as if After Effects ever updated or extended their binary format the whole house of card would come crumbling down. So instead I took a closer look at the binary file.

# RIFX

The first order of business was to open the `.aep` in a hex editor. For this I used the [hexdump for VSCode](https://marketplace.visualstudio.com/items?itemName=slevesque.vscode-hexdump) extension for Visual Studio Code. This is what the beginning of the file looked like:

![hexdump of an AE .aep](../../../assets/img/gradients-in-after-effects/hexdump.png)

The most interesting part of this was the first 4 characters: `RIFX`. Some quick googling said that `RIFX` is a variant of the `RIFF` format in which chunk sizes are stored in big-endian format. The use of big-endian was likely due to the ancient roots of After Effects where it ran on PowerPC machines.

Looking up the specs for the `RIFF` format yielded [this page](https://www.fileformat.info/format/riff/corion.htm). Luckily the format was extremely simple, and could be surmised in two tables:

| Offset | Bytes | Type  | Description |
|--------|-------|-------|-------------|
| 0000h  | 4     | char  | The chunk type. This can be anything and identifies what the chunk contains. |
| 0004h  | 4     | dword | The size of the chunk content. This does not include the size of the chunk type and the size of the size of the chunk content. |
| 0008h  | ???   | ???   | The chunk data. The previous field indicates how long this section is. |

Chunks of type `LIST`, `RIFF` or `RIFX` use the 'subblock' format, which adds a subblock type:

| Offset | Bytes | Type  | Description |
|--------|-------|-------|-------------|
| 0000h  | 4     | char  | The chunk type. For subblocks this is `LIST`, `RIFF` or `RIFX`. |
| 0004h  | 4     | dword | The size of the chunk content. This does not include the size of the chunk type and the size of the size of the chunk content. |
| 0008h  | 4     | char  | The subblock type. This identifies the type of subblock. |
| 000Ch  | ???   | ???   | The chunk data. Note that the size of this section is `(size of chunk content - 4)` since 4 bytes are used up by the subblock type. |

Subblocks can contain other subblocks, and this can be used to create tree structures.

Finally, the spec requires that all chunks start on a 2 byte boundary. This means that if the chunk size is not a multiple of 2 then 1 needs to be added to the data pointer before reading from the next block.

Fun fact: the subblock type of `RIFX` chunk in the `.aep` file is `Egg!`, which was the code name of the [first version](https://en.wikipedia.org/wiki/Adobe_After_Effects#History) of After Effects.

Reading RIFF data in a hex editor isn't really easy, so I wrote a quick C++ parser to dump the file's data. This gave an output like this:

```
<svap> (4 bytes) '@0', hex: 0x0B400E30, dec: 188747312
<head> (20 bytes) '', hex: 0x005D00040B400E3080000000000000130000004B
<nhed> (32 bytes) '', hex: 0x0000000000000005000101001E100200010000000B692F920000000102755AD2
<nnhd> (40 bytes) '', hex: 0x0000000000000005000101000000001E0000001002000000000100000B692F920000000102755AD2
<adfr> (8 bytes) '@\347p', hex: 0x40E7700000000000
<Pefl> LIST (4 bytes)
<qtlg> (1 bytes) '', hex: 0x00, dec: 0
<gpuG> LIST (48 bytes)
  <Utf8> (36 bytes) 'be93941a-7488-4117-8a46-7e3596950307', hex: 0x62653933393431612D373438382D343131372D386134362D376533353936393530333037
<sfnm> LIST (30 bytes)
  <Utf8> (6 bytes) 'Solids', hex: 0x536F6C696473
  <sfid> (4 bytes) '', hex: 0x0C000000, dec: 201326592
<acer> (1 bytes) '', hex: 0x01, dec: 1
```

This looks _very_ similar to the XML output:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<AfterEffectsProject xmlns="http://www.adobe.com/products/aftereffects" majorVersion="1" minorVersion="0">
	<svap bdata="0b400e30"/>
	<head bdata="005d00040b400e30800000000000000d0000009b"/>
	<nhed bdata="0000000000000005000101001e1002000000000039954210000000014de26c80"/>
	<nnhd bdata="0000000000000005000101000000001e00000010020000000000000039954210000000014de26c80"/>
	<adfr bdata="40e7700000000000"/>
	<Pefl>
	</Pefl>
	<qtlg bdata="00"/>
	<gpuG>
		<string>be93941a-7488-4117-8a46-7e3596950307</string>
	</gpuG>
	<sfnm>
		<string>Solids</string>
		<sfid bdata="00000000"/>
	</sfnm>
        ...
```

Because of the strong resemblance, it was easier to read from the XML file when trying to decypher the format. In some cases string data was present as strings (such as the `string` XML tag, which represented a `Utf8` chunk in the RIFF data), and sometimes the strings were encoded as hexadecimal in `bdata` values. Interesting parties can look at [this file](../../../assets/aefiles/DemoGradients.aepx.txt) for an example of what the After Effects XML format looks like.

While reading `RIFF` data is easy, it isn't obvious what each field means. What does `gpuG` mean? Maybe a GPU GUID, given what its subblocks are? What about `sfnm`? The next step was finding how to get to the data that I needed.

# Composition and Layer ID

In the Javascript API, compositions are identified by ID, and layers are identified by index. In order to extract the gradient color data for the correct layer, those items need to be identified in the data.

Compositions use the `Item` tag instead of the expected `Comp`, as `Item` is what compositions are called in the AE C++ API. Note that Items aren't _just_ compositions, they're really any source present in the project. To get the composition ID, we need to dig into the chunks of `Item` LISTs. The process followed was to create an `.aep` file with multiple identical compositions (but with different names). The XML output then looks like this:

```xml
<Item>
        <idta bdata="000400000000000000000000000000000000002c00000020000000000000000000000000000000000000000000000000000000000000000000000f000000000000000000000000000000000000000000d9283eac"/>
        <string>Example Composition 2</string>
        ...
</Item>
...
<Item>
        <idta bdata="000400000000000000000000000000000000003700000020000000000000000000000000000000000000000000000000000000000000000000000f000000000000000000000000000000000000000000d9283eac"/>
        <string>Example Composition 3</string>
        ...
</Item>
...
<Item>
        <idta bdata="000400000000000000000000000000000000004200000020000000000000000000000000000000000000000000000000000000000000000000000f000000000000000000000000000000000000000000d9283eac"/>
        <string>Example Composition 4</string>
        ...
</Item>
```

Looking carefully at the `idta` chunk's data, we can see that there is one varying part of the binary data. Converting to binary, and knowing that the composition IDs in the AE SDK are 4 bytes large, we can determine that the composition ID is bytes 16 to 19 of the `idta` chunk.

Locating layers is easy, as those use the obvious `Layr` chunk name:

```xml
<Layr>
        <ldta bdata="0000000f000200000000000100000000000078000000000000007800000e100000007800000100070000000e00000000000000000000000000000001000100004578616d706c6520536f6c69640000000000000000000000000000000000000000000002000000000000000000000001000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000"/>
        <string>Example Solid</string>
        <tdgp>
                <tdsb bdata="00000001"/>
                <tdsn>
                        <string/>
                </tdsn>
                <tdmn bdata="414442452054696d652052656d617070696e67000000000000000000000000000000000000000000"/>
                ...
```

Determining the index is therefore simple: just count the `Layr` chunks in the parent `Item` composition. The order of the `Layr` chunks matches the order of the layers within a composition in After Effects. The only thing of note is that in After Effects layer indices are 1-based rather than 0-based.

# Identifying Shape Components

Unlike compositions, shape components are not identified by ID. Instead, the tree structure of the shapes within the shape layer is represented directly in the XML/RIFF data. To uniquely identify a shape component, we need to store an index for each group and for the shape itself. For example, given the following shape layer:

```
Shape Layer
  Group 1
     Group 2
        Rectangle Path 2
        Gradient Stroke 1
     Rectangle Path 1
     Gradient Fill 1
```

The Gradient Fill could be identified as `0,2` where 0 is the index of `Group 1` within the shape layer, and 2 is the index of the `Gradient Fill 1` within `Group 1`. The gradient stroke would be identified as `0,0,1`.

In the XML data there is also a 'root group' used to hold the multiple shapes that a shape layer can hold. Since every shape layer has a single root group, it isn't necessary to include it to identify shapes within a shape layer.

Finally, in the XML, each shape is contained in a `tdgp` chunk. However shape groups also contain an extra embedded `tdgp` chunk -- this must be accounted for when searching for 'our' gradient shape. Again, this was determined by trial-and-error by creating multiple shape layers with varying shape/group content.

# Accessing the AEP File

In the bodymovin plugin, the AEP file is accessed through the `app.project` object. However this only works if the file has been saved before, which in our case isn't necessarily true: users can use the Preview Tool on files whose modifications haven't been saved yet, or on projects that have never been saved.

To work around this issue, the You.i Engine After Effects plugin first checks if the project has unsaved modification using the `AEGP_ProjectIsDirty` AE SDK function. If the file is unmodified, then the project's `.aep` is parsed directly.

If the project _does_ have modifications, or if the project has never been saved, then the project must first be saved. Since the user may not appreciate the project being saved behind their backs, the plugin instead saves a copy of the project to a temporary directory. Once the `.aep` has been parsed, the new file is deleted. Unfortunately this has the side-effect of adding this temporary file to the 'recent projects' list in After Effects.

While testing it was found out that After Effects can open XML files directly: if the current project's filename is an `.aepx` file, then we also write an `.aep` to a temporary folder. If using `.aepx` files is common enough, we may later add support for parsing those files directly.

Within the plugin, the `.aep` file is only parsed once and is cached to avoid needing to re-save and re-parse the file for each shape layer and for each individual gradient.

# Parsing (For Real)

To parse `.aep` files in the You.i Engine After Effects plugin, a more robust parser was needed than the one used to reverse-engineer the file format. This new parser needed to make it easy to get to the chunks we're interested in, and needed to be resilient to misformed (or incomplete) files.

For simplicity, the parser was written to read the whole file into memory and then parse it. Memory isn't really an issue here as the plugin (and thus the parser) is only meant to run on desktop platforms.

Follows is the API for the parser:

```c++
class RIFFParser
{
public:
    struct Chunk
    {
        CYIString type;
        size_t size = 0; // Excludes the size and the type
        std::vector<Chunk> chunks;
        CYIString data;

        bool IsEmpty() const;
        std::vector<const Chunk *> GetChunksNamed(const CYIString &name) const;
        std::vector<const Chunk *> GetChunksNamedRecursive(const CYIString &name) const;
        const Chunk &GetFirstChunkNamed(const CYIString &name) const;

    private:
        void GetChunksNamedRecursive(const CYIString &name, std::vector<const Chunk *> *pResult) const;
    };

    static uint32_t ReadUint32(const char *pData, bool usesLittleEndian = true);
    static int32_t ReadInt32(const char *pData, bool usesLittleEndian = true);

    static Chunk ParseFromStream(std::ifstream *pStream);
    static Chunk ParseFile(const CYIString &filename);
};
```

The parsing functions return a single `Chunk` object, which contains the chunks from the parsed file. Each chunk contains a type string and either a list of chunks or raw data for the chunk. The 'size' field wasn't really needed for users of `Chunk` but made parsing easier.

The parser supports both `RIFF` and `RIFX` formats. Depending what the type of the first parsed chunk is, the size fields are read either as little-endian or big-endian. In the case of `.aep` files, the format is `RIFX` and thus all integer values must be read as big-endian.

For accessing parsed chunks, the `GetChunksNamed`, `GetChunksNamedRecursive` and `GetFirstChunkNamed` functions can be used. Those functions either return an empty vector or an empty chunk when the requested chunk(s) could not be found -- this makes it easy to chain calls like this:

```c++
const RIFFParser::Chunk &group = layer.GetFirstChunkNamed("tdgp").GetFirstChunkNamed("tdgp");
```

Iterating over components of a chunk is also easy:

```c++
for (const auto *pLayer : parsed.GetChunksNamed("Layr"))
{
    ...
}
```

The parsed data for a composition looks like this in the Xcode debugger:

![Chunks in Xcode](../../../assets/img/gradients-in-after-effects/parsed_data_in_xcode.png)

# The Final Hurdle

After doing all this parsing and locating the appropriate composition, layer, and shape component, there is one final problem that needs to be addressed. This is what the chunks for a gradient shape look like:

![Gradient Colors in Xcode](../../../assets/img/gradients-in-after-effects/gradient_colors_in_xcode.png)

Locating the color data is simple enough: once the gradient shape is found, look for the `GCst` chunk, then for the `GCky` chunk, and finally for its `Utf8` chunk. What we find is somewhat surprising (or would be surprising if we didn't already see it in the XML file): the gradient color data is stored as XML _within_ the RIFX file. This looks especially ridiculous in the XML AE files, where the XML gradient color data is stored as HTML-encoded strings within the file's XML. Why Adobe decided to do that is unknown -- most likely they already supported gradients in some other product of theirs and just saved the data in the same way.

In any case, parsing XML data is easy: just use any of the thousands of available XML libraries. You.i Engine includes [tinyxml2](http://www.grinninglizard.com/tinyxml2/index.html) internally, so that's what we used for parsing the gradient color data.

The XML data for the gradients color is somewhat more verbose than it needs to be:

```xml
<?xml version='1.0'?>
    <prop.map version='4'>
        <prop.list>
            <prop.pair>
                <key>Gradient Color Data</key>
                <prop.list>
                    <prop.pair>
                        <key>Alpha Stops</key>
                        <prop.list>
                            <prop.pair>
                                <key>Stops List</key>
                                <prop.list>
                                    <prop.pair>
                                        <key>Stop-0</key>
                                        <prop.list>
                                            <prop.pair>
                                                <key>Stops Alpha</key>
                                                <array>
                                                    <array.type><float/></array.type>
                                                    <float>0</float>
                                                    <float>0.5</float>
                                                    <float>1</float>
                                                </array>
                                                </prop.pair>
                                            </prop.list>
                                        </prop.pair>
                                        <prop.pair>
                                            <key>Stop-1</key>
                                            <prop.list>
                                            ...
```

The structure of the XML is a 1-to-1 representation of the data in the 'gradient colors' dialog box in After Effects:

![Gradient Colors Dialog in After Effects](../../../assets/img/gradients-in-after-effects/gradient_colors_dialog.png)

The XML also contains superfluous information such as the number of stops (why not just count the number of entries?)

The final step to parse the color data was to parse this XML to access the gradient color data. One interesting thing about this data is that it has separate gradient stops for alpha and for colors. This is reflected in the After Effects dialog -- Adobe must have though it was important for designers to be able to control the alpha and the colors independently. Also interesting is that even if alpha can't be specified for color stops, there is still an alpha entry present in the XML data for color stops.

The meaning of the various float values within the XML is not obvious (why weren't separate tags created for each?), but can be determined by using unique values in the After Effects gradient colors dialog and seeing where each of those value show up in the XML data. In the case of a single 'Stops Alpha' entry, the first float is the location of the stop, the second value is the midpoint for the stop vs the next stop, and the third value is the opacity for the stop itself. Each value ranges from 0 to 1. For color stops, there are 6 values: the location of the stop, the midpoint value, the red, green and blue values, and finally an unused alpha component (always set to 1).

One last oddity: when creating a 'new' gradient stroke or gradient fill in After Effects, the gradient defaults to a white-to-black gradient. If the colors are not modified by the user, After Effects skips writing a `GCst` chunk to the file. The absence of that block must be interpreted as a gradient with a white color stop at 0% and a black color stop at 100% (with 100% opacity alpha stops at the beginning and end.)

# Rendering in You.i Engine

I'll keep this section brief as it doesn't really relate to reading the gradient data.

In the plugin, all available gradient data is written out to Protobuf files. We also export data that we know won't be supported by You.i Engine (such as radial gradient highlights and gradient color stop midpoints).

In the engine, only the first and last color/alpha stops are used, as the `NanoVG` rendering backend doesn't support having multiple color stops.

Once in the engine, the gradient data (and the shape data) is passed to a `CYIVectorCanvasNode` object, which then translates that data into `NanoVG` rendering commands.

![Render Commands in CYIVectorCanvasNode](../../../assets/img/gradients-in-after-effects/commands_in_vectorcanvasnode.png)

# In The Future

Eventually, support for animating shape layer components will be added to You.i Engine. This poses a new problem, as gradient colors can also be animated. At that point we will likely need to revisit how gradient color is accessed and stored in the context of an animation.
