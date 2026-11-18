# Memory use when loading a city or changing graphics settings

## What prompted the change

On Windows with an 8 GiB RTX 2060 SUPER, loading the Los Angeles area ended with a failed 160 MiB host allocation. After limiting concurrent building jobs, another session reached the same area and ran longer. This time wgpu reported an out-of-memory error while the player was changing graphics settings. The logs identify neither the object behind the first allocation nor the GPU resource that failed in the second run.

Two parts of the code explain why memory demand can peak during these operations. A worker can finish generating a city tile while its mesh still occupies RAM awaiting transfer. Once uploaded, tiles continue to occupy GPU memory as the camera moves. Changing resolution or shadow quality also requires new textures while the renderer still holds the ones used for the previous frame.

## Why the limits belong in the scenery loader

The loader knows which tiles are closest to the aircraft and when a transfer actually finishes. It now counts a building tile from dispatch through its final upload. A limit of two allows one tile to be prepared while another uploads, and leaves workers available for terrain.

Tile sizes vary with building density, so a count alone gives little control over resident geometry. Before allocating GPU buffers, the loader adds the incoming buffers and texture atlas to the estimated payload of resident tiles and uploads in progress. When this would exceed 768 MiB, it first makes room from farther tiles or defers the incoming tile. A distance margin reduces repeated eviction and rebuilding near the cutoff. In a dense city, this can delay the appearance of distant buildings.

Two jobs and 768 MiB are initial tuning choices. The byte budget covers the OpenStreetMap building tile payload. Terrain imagery, render targets, shadows, driver overhead and other applications also consume memory. The budget leaves room for those consumers but cannot predict the memory available on every GPU.

## Why use wgpu for settings changes

FableFlight already creates its textures, views and bind groups through wgpu. Its error scopes let the renderer associate an allocation refusal with the particular setting being applied, using the same code across graphics backends. A single device-wide error handler would lack the context to choose which setting and resources to retain.

When render scale or shadow size changes, the renderer waits for outstanding GPU work, then creates the replacements inside scoped error handling. The old resources stay usable until the new set succeeds. If the allocation fails, the renderer retains the previous value for that setting and reports it to the player. Validation errors without an allocation failure still surface as programming errors. Dragging the resolution slider previews the size and commits it on release, avoiding a fresh set of textures at every intermediate position.

## What was verified

On the affected GPU, before adapting the patch to the current settings menu, an offline KLAX session switched from High to Medium to Ultra and returned to the simulation. A separate release run with cached Los Angeles scenery completed 165 seconds at Medium and exited normally. The two graphics preset tests passed on the adapted branch. These checks exercised successful settings changes. Forced allocation refusal, long online flights and tuning on other GPUs remain open validation cases.
