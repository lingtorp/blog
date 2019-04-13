# Tile based level representation in Unity

Sometimes you need a quick and dirty way to store and create levels based on a set of tiles. Sure, Unity offers native support for tilemaps (since 2018.2 or .3) but often simple approaches works until a more solid workflow can be established. This is the case I found while developing Socani, a 2D tile-based Sokoban inspired casual puzzle game.

## Pixel maps
The idea here is simple. You simply store the relevant tiles in layers of textures where each pixel is one tile. For example if the level is 10x10 then one 10x10 pixel texture describes one layer of that level. Stacking tiles is the same stacking textures which is usually supported in image editing programs as layers. Therefore it is useful to be able to export all the layers of a pixel image as seperate images.

First you setup a color mapping between a set of RGBA colors and a set of prefabs in your game. This is represented as an array of all the defined colors and their respective prefab.

Here is how that palette looks in Unity.

![](../images/tilemappings.png "Tilemappings between images and prefabs.")

Next, we need a container that represents on level. This level is composed of multiple textures in a layered format similiar to a cake.

![](../images/tilelayers.png)

**Note:** In Unity you need to mark the textures as *Read/Write Enable* in order to actually read the pixels. 

Here is where the glue code needs to be written that actually reads the pixels from the textures and instantiates the prefabs. This is very game specific but in Socani I simply lookup the pixels color and then add a reference to the matching prefab in a Dictionary where Vector2's are mapped to lists of Tiles. The prefabs are instantiated by the caller to the level loading function due to things like scaling the level and so on.

```cs
  // Loads the tile layers in to a Dictionary with the tile position as key and a list of tiles as the value
public Dictionary<Vector2Int, List<GameObject>> Load(GameBoard gameboard, ref Dictionary<Vector3Int, Color> tileMappings) {
  Dictionary<Vector2Int, List<GameObject>> board = new Dictionary<Vector2Int, List<GameObject>>();

  int maxWidth  = 0; // Dimensions of the loaded lvl in tiles
  int maxHeight = 0;

  if (tiledLevelTMXData.Length > 0) {
    Debug.Log("Loading TMX");
    return loadTMXFile(tiledLevelTMXData);
  }

  for (int z = 0; z < tileLayers.Length; z++) {
    Texture2D tileLayer = tileLayers[z];

    if (tileLayer.width  > maxWidth)  { maxWidth = tileLayer.width; }
    if (tileLayer.height > maxHeight) { maxHeight = tileLayer.height; }

    for (int x = 0; x < tileLayer.width; x++) {
      for (int y = 0; y < tileLayer.height; y++) {
        // Generate a new tile
        Color color = tileLayers[z].GetPixel(x, y);
        if (color.a == 0.0f) { continue; }

        var position = new Vector2Int(x, y);
        if (!board.ContainsKey(position)) {
          board[position] = new List<GameObject>(2);
        }

        bool foundMatchingColor = false;
        for (int i = 0; i < gameboard.mappings.Length; i++) {
          TileMapping tileMapping = gameboard.mappings[i];
          if (color.Equals(tileMapping.color)) {
            foundMatchingColor = true;

            board[position].Add(tileMapping.prefab);
            // Construct level variables
            if (tileMapping.prefab.GetComponent<Coin>()) {
              numCoins++;
            }

            tileMappings[new Vector3Int(x, y, board[position].Count - 1)] = tileMapping.color;
          }
        }

        if (!foundMatchingColor) { Debug.Log(string.Format("Did not find matching color for color: {0}", color)); }
      }
    }
  }
  dimensions = new Vector2Int(maxWidth, maxHeight);
  return board;
}
```

Next up we need a way to actually create the pixel image that will be our level. 

My pixel editor of choice is [Aseprite](https://www.aseprite.org/) which is a nice retro-styled pixel editor with quite a few advanced features as well.

Aseprite supports saving an entire palette so that is useful. Lets look how one level might look in Aseprite.

![](../images/aseprite-level7.png "Sooo confusing..")

Notice the palette the left matching the one defined in Unity and all the different colors are in their own layers. Now we need to export this image but in a way such that each layer will be its own image. For this purpose we need to use Aseprite's command line interface in a .bat script (if we are on Windows that is).

```bat
D:
cd \SteamLibrary\steamapps\common\Aseprite 
@set ASEPRITE=Aseprite.exe
@set INPUT_NAME=level7
@set OUTPUT_NAME=%INPUT_NAME%
%ASEPRITE% -b --split-layers C:\Users\Alexander\repos\Socani\Assets\levels\%INPUT_NAME%\%INPUT_NAME%.aseprite --save-as %OUTPUT_NAME%-{layer}.png
timeout /t -1
```

After this we are left with multiple textures each corresponding to each layer in the image. Now we simply import them all into Unity and drag them into the specified level prefab and **BOOM**! We are done!

### Remarks
The method works fine for small, simple levels but when the complexity of the levels increase this method does **not** scale well. Just one look at the image of level7 in Aseprite should tell you the amount of confusion that I had to endure whilst creating that level. I cant even begin to describe the amount of times I lookup the color mapping since I had forgot which tint of yellow was the Crate prefab and which was the Coin one ...

It was obvios I needed an easier, faster, more visual approach to the level creating pipeline. Enter [Tiled](https://www.mapeditor.org/).

**Note:** There is a SuperTiled.unitypackage out there on the interwebs. I tried it, got it to work. It simply did not suit my simplistic needs.

## Tiled --> XML --> Unity 
[Tiled](https://www.mapeditor.org/) is an awesome QT-based tilemap editor that can both store tileset (atlas of sprites) and a grid of sprite in a tilemap but also export these into lovely XML (.tmx and .tsx respectively). The thing that makes Tiled magical is its ability to add custom data to the tiles in the tileset and thus pass extra information to the application consuming the level information (us!).

![](../images/tiled-example.png "Soooo muuuuch bettteeeerr")

Here we can import our set of sprites into a tileset, add custom data (see the left "Custom Properties"), paint our level in different layers just as in Aseprite and then just save it to XML. 

The trick here is that the color mapping in Unity is gone. Now we simply specify the prefab name for each tile in the tileset. In Unity we simply read this property, lookup the corresponding prefab and load it! 

```cs
  static Dictionary<int, GameObject> loadTileset(string filepath) {
    XmlDocument xmlDoc = new XmlDocument();
    xmlDoc.Load("Assets/Resources/levels/levels" + "/" + filepath); // NOTE(2): Tiled uses relative filepaths ..
    Dictionary<int, GameObject> tileset = new Dictionary<int, GameObject>();
    foreach (XmlNode childNode in xmlDoc.ChildNodes) {
      if (childNode.Name == "tileset") {
        foreach (XmlNode tile in childNode.ChildNodes) {
          if (tile.Name == "tile") {
            int id = int.Parse(tile.Attributes.GetNamedItem("id").Value);
            foreach (XmlNode property in tile.FirstChild.ChildNodes) { // Assumption
              if (property.Attributes.GetNamedItem("name")?.Value == "prefab_name") {
                string prefabName = property.Attributes.GetNamedItem("value").Value;
                Debug.Log("prefabs/" + prefabName);
                tileset[id] = Resources.Load<GameObject>("prefabs/" + prefabName);
                if (tileset[id] == null) {
                  Debug.LogError("Lets start to question our life choices");
                }
                Debug.Log("Loaded " + "prefabs/" + prefabName + " with id =" + id);
              }
            }
          }
        }
      }
    }
    return tileset;
  }

  static Dictionary<Vector2Int, List<GameObject>> loadTMXFile(string tmxFile) {
    XmlDocument xmlDoc = new XmlDocument();
    xmlDoc.Load(tmxFile);
    foreach (XmlNode childNode in xmlDoc.ChildNodes) {
      if (childNode.Name == "map") {
        int width = int.Parse(childNode.Attributes.GetNamedItem("width")?.Value);
        int height = int.Parse(childNode.Attributes.GetNamedItem("height")?.Value);
        dimensions = new Vector2Int(width, height);
        Dictionary<int, GameObject> tileset;
        if (childNode.FirstChild.Name == "tileset") {
          string tileSetPath = childNode.FirstChild.Attributes.GetNamedItem("source").Value;
          tileset = loadTileset(tileSetPath);
        } else {
          Debug.LogError("Tileset missing from level .tmx");
          return null;
        }
        Dictionary<Vector2Int, List<GameObject>> board = new Dictionary<Vector2Int, List<GameObject>>();
        foreach (XmlNode layer in childNode.ChildNodes) {
          if (layer.Name == "layer") {
            string tileLayer = layer.FirstChild.InnerText; // CSV 
            string[] tiles = tileLayer.Split(',');
            for (int x = 0; x < width; x++) {
              for (int y = 0; y < height; y++) {
                // NOTE(1): .tmx, 0 = empty thus all indices are + 1 and need to be decremented
                int tileIdx = int.Parse(tiles[x * width + y]) - 1;
                if (tileIdx <= -1) { continue; }
                var position = new Vector2Int(x, y);
                if (!board.ContainsKey(position)) {
                  board[position] = new List<GameObject>(2);
                }
                board[position].Add(tileset[tileIdx]);
              }
            }
          }
        }
        return board; // XML parsing succeeded
      }
    }
    return null; // XML parsing failed
  }
```

There are a couple of gotchas with this approach but other than that it is simply a standard XML parsing scheme!
1. Tiled uses 0 to represent 'empty' tiles in the level which means that tile 25 in the level is actually tile 24 in the tileset.
2. I think this is a huge positive and that is that all the filepaths are relative.
3. Unity's Resources.Load function is **ONLY** able to load prefabs and what not from the '*Resources*' folder. I therefore had to move everything in the Unity project from '*Assets/*' to '*Assets/Resources/*'.

Here is how the level looks in XML from Tiled ...
```xml
<?xml version="1.0" encoding="UTF-8"?>
<map version="1.2" tiledversion="1.2.3" orientation="orthogonal" renderorder="right-down" width="7" height="7" tilewidth="128" tileheight="128" infinite="0" nextlayerid="4" nextobjectid="1">
 <tileset firstgid="1" source="../kenney_sokoban.tsx"/>
 <layer id="1" name="Tile Layer 1" width="7" height="7">
  <data encoding="csv">
81,81,81,81,81,81,81,
81,81,81,81,81,81,81,
81,81,81,81,81,81,81,
81,81,81,81,81,81,81,
81,81,81,81,81,81,81,
81,81,81,81,81,81,81,
81,81,81,81,81,81,81
</data>
 </layer>
 <layer id="2" name="Walls" width="7" height="7">
  <data encoding="csv">
71,71,71,71,71,71,71,
71,0,0,0,0,0,71,
71,0,0,0,0,0,71,
71,0,0,0,0,0,71,
71,0,0,0,0,0,71,
71,0,0,0,0,0,71,
71,71,71,71,71,71,71
</data>
 </layer>
 <layer id="3" name="Crates" width="7" height="7">
  <data encoding="csv">
0,0,0,0,0,0,0,
0,0,0,0,0,0,0,
0,0,0,26,0,0,0,
0,0,26,0,26,0,0,
0,0,0,0,0,0,0,
0,0,0,0,0,0,0,
0,0,0,0,0,0,0
</data>
 </layer>
</map>
```

... and here is the tileset.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<tileset version="1.2" tiledversion="1.2.3" name="kenney_sokoban" tilewidth="128" tileheight="128" tilecount="75" columns="0">
 <grid orientation="orthogonal" width="1" height="1"/>
  ... 
 <tile id="25">
  <properties>
   <property name="prefab_name" value="Crate"/>
  </properties>
  <image width="128" height="128" source="../assets/Crates/crate_02.png"/>
  ...
</tileset>
```

## Closing Remarks
I expect a huge productive boost from the simple fact that I can now see the levels as they will be seen in the game and thus able to mentally play them and test them before even switching to Unity. Hopefully the custom properties system of Tiled enables other types of gameplay such as teleporters and what not! I am certainly looking forward to having this be the way I create levels. :)

