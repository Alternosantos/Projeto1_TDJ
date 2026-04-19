```
+======================================================================+
|......................................................................|
|.....███╗...███╗██╗███╗...██╗███████╗██████╗.██╗██████╗.████████╗.....|
|.....████╗.████║██║████╗..██║██╔════╝██╔══██╗██║██╔══██╗╚══██╔══╝.....|
|.....██╔████╔██║██║██╔██╗.██║█████╗..██║..██║██║██████╔╝...██║........|
|.....██║╚██╔╝██║██║██║╚██╗██║██╔══╝..██║..██║██║██╔══██╗...██║........|
|.....██║.╚═╝.██║██║██║.╚████║███████╗██████╔╝██║██║..██║...██║........|
|.....╚═╝.....╚═╝╚═╝╚═╝..╚═══╝╚══════╝╚═════╝.╚═╝╚═╝..╚═╝...╚═╝........|
|......................................................................|
+======================================================================+
```

[![Discord](https://img.shields.io/discord/1165553796223602708?style=flat-square&logo=discord&logoColor=white&label=Discord&color=%237289DA&link=https%3A%2F%2Fdiscord.gg%2FnU63sFMcnX)](https://discord.gg/nU63sFMcnX)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=flat-square)](LICENSE.txt)
![Language](https://img.shields.io/badge/C%23-94.3%25-blue?style=flat-square)
![HLSL](https://img.shields.io/badge/HLSL-5.7%25-purple?style=flat-square)

# MineDirt

**MineDirt** is a Minecraft-inspired voxel engine built with [MonoGame](https://www.monogame.net/), written in C# and HLSL. It focuses on high-performance chunk rendering, procedural terrain generation, and efficient voxel meshing techniques.

---

## Features

- **Procedural Terrain Generation** — Heightmap-based world generation using FastNoiseLite for natural-looking landscapes
- **Greedy Meshing** — Optimized mesh generation that merges adjacent faces to minimize draw calls and GPU load
- **Visibility Culling** — Chunk-level and face-level culling to avoid rendering geometry that isn't visible to the player
- **Custom Shaders** — HLSL shaders for rendering the voxel world with lighting and effects
- **ImGui Debug UI** — Integrated Dear ImGui overlay via MonoGame.ImGuiNet for real-time debugging and diagnostics
- **Chunk System** — World divided into manageable chunks for efficient loading and rendering

---

## Getting Started

### Prerequisites

- [.NET SDK](https://dotnet.microsoft.com/download) (version compatible with MonoGame 3.8+)
- [MonoGame](https://www.monogame.net/downloads/) framework
- A desktop OS: Windows, macOS, or Linux

### Installation

1. Clone the repository:
   ```bash
   git clone https://github.com/vycdev/MineDirt.git
   cd MineDirt
   ```

2. Open the solution in your IDE:
   ```bash
   # Visual Studio
   start MineDirt.sln

   # Or via CLI
   dotnet build MineDirt.sln
   ```

3. Run the project:
   ```bash
   dotnet run --project MineDirt
   ```

---

## Project Structure

```
MineDirt/
├── MineDirt/           # Main game project (C# + HLSL)
├── MonoGame.ImGuiNet/  # ImGui integration for MonoGame (debug UI)
├── MineDirt.sln        # Visual Studio solution file
└── LICENSE.txt
```

---

## Controls

> Controls may vary — check in-game or refer to source for keybindings.

| Action | Key / Input |
|---|---|
| Move | `W A S D` |
| Look | Mouse |
| Jump | `Space` |
| Toggle debug UI | `F3` or ImGui overlay |

---

## Notable Functions

These are some notable functions that are already implemented 

Handles player camera movement
```
public bool HandleMovement(GameTime gameTime)
    {
        Vector3 moveDirection = Vector3.Zero;

        Matrix yawRotation = Matrix.CreateRotationY(yaw);

        Vector3 forwardPlanar = Vector3.Transform(Vector3.Forward, yawRotation);
        Vector3 rightPlanar = Vector3.Transform(Vector3.Right, yawRotation);

        if (kbState.IsKeyDown(Keys.W))
            moveDirection += forwardPlanar;

        if (kbState.IsKeyDown(Keys.S))
            moveDirection -= forwardPlanar;

        if (kbState.IsKeyDown(Keys.A))
            moveDirection -= rightPlanar;

        if (kbState.IsKeyDown(Keys.D))
            moveDirection += rightPlanar;

        if (kbState.IsKeyDown(Keys.LeftShift))
            moveDirection += Vector3.Down;

        if (kbState.IsKeyDown(Keys.Space))
            moveDirection += Vector3.Up;

        if (moveDirection.LengthSquared() > 0.001f)
        {
            Position += Vector3.Normalize(moveDirection) * GetMovementSpeed() * (float)gameTime.ElapsedGameTime.TotalSeconds;
            return true;
        }

        return false;
    }
```

Generates random terrain chunks
```
public void GenerateTerrain()
```

Handles block breaks 
```
public static void BreakBlock(Vector3 position)
    {
        if (Chunks.TryGetValue(position.ToChunkPosition(), out Chunk chunk))
        {
            int index = position.ToChunkRelativePosition().ToIndex();
            if (chunk.Blocks[index].Type == BlockType.Air)
                return;

            chunk.Blocks[index] = default;
            chunk.BlockCount--;
            MeshThreadPool.EnqueueTask(() =>
            {
                ChunkMeshData meshData = chunk.GenerateMeshData();

                ChunkBufferGenerationQueue.Enqueue(() => chunk.GenerateBuffers(meshData));
            });

            if (chunk.TryGetBlockChunkNeighbours(index, out List<Vector3> chunkNbPositions))
            {
                foreach (Vector3 chunkNbPos in chunkNbPositions)
                {
                    if (Chunks.TryGetValue(chunkNbPos, out Chunk chunkNb))
                    {
                        MeshThreadPool.EnqueueTask(() =>
                        {
                            ChunkMeshData meshData = chunkNb.GenerateMeshData();

                            ChunkBufferGenerationQueue.Enqueue(() => chunkNb.GenerateBuffers(meshData));
                        });
                    }
                }
            }
        }
    }
```

Handles bblock placement 
```
public static void PlaceBlock(Vector3 position, Block block)
    {
        if (position.Y > Chunk.Height || position.Y < 0)
            return;

        if (Chunks.TryGetValue(position.ToChunkPosition(), out Chunk chunk))
        {
            int index = position.ToChunkRelativePosition().ToIndex();
            if (chunk.Blocks[index].Type != BlockType.Air)
                return;

            chunk.Blocks[index] = block;
            chunk.BlockCount++;
            MeshThreadPool.EnqueueTask(() =>
            {
                ChunkMeshData meshData = chunk.GenerateMeshData();

                ChunkBufferGenerationQueue.Enqueue(() => chunk.GenerateBuffers(meshData));
            });

            if (chunk.TryGetBlockChunkNeighbours(index, out List<Vector3> chunkNbPositions))
            {
                foreach (Vector3 chunkNbPos in chunkNbPositions)
                {
                    if (Chunks.TryGetValue(chunkNbPos, out Chunk chunkNb))
                    {
                        MeshThreadPool.EnqueueTask(() =>
                        {
                            ChunkMeshData meshData = chunkNb.GenerateMeshData();

                            ChunkBufferGenerationQueue.Enqueue(() => chunkNb.GenerateBuffers(meshData));
                        });
                    }
                }
            }
        }
    }
```



## Roadmap / Current Status

MineDirt is currently in **alpha** (v0.2.0-alpha). Active development is ongoing. Planned or in-progress features may include:

- Block placement and destruction
- Inventory system
- More biomes and terrain features
- Lighting system
- Multiplayer support

---

## Credits

- **FastNoiseLite** — Noise library for terrain generation: [Auburn/FastNoiseLite](https://github.com/Auburn/FastNoiseLite)
- Terrain generation concepts: [YouTube — Terrain Generation](https://www.youtube.com/watch?v=CSa5O6knuwI)
- Voxel engine design: [Let's Make a Voxel Engine](https://sites.google.com/site/letsmakeavoxelengine/home)
- High performance voxel engine techniques: [nickmcd.me](https://nickmcd.me/2021/04/04/high-performance-voxel-engine/)
- Visibility culling (from MCPE creators): [tomcc.github.io](https://tomcc.github.io/2014/08/31/visibility-2.html)
- Greedy meshing: [fluff.blog](https://fluff.blog/2023/04/24/greedy-meshing-visually.html)

---

## Contributing

Contributions are welcome! Feel free to open issues or submit pull requests. For larger changes, consider discussing them on the [Discord server](https://discord.gg/nU63sFMcnX) first.

---

## License

This project is licensed under the [MIT License](LICENSE.txt).

##  Original Repo

- [This project was created by vycdev and guyfromtheforest ](https://github.com/vycdev/MineDirt)

## Students who worked onn this projects readme

-a28316 Alberto Santos
-a28616 Paulo Aguiar