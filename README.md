# Garden Zen

See your Obsidian vault as a zen garden. Every note is a plant that grows as you write and link, every folder is an island in raked sand.

## How it works

- **Plants grow** from seed to tree with the words a note holds and the links it gives and receives (backlinks weigh double).
- **Islands are planets.** Each folder (or type, tag…) is its own island. The note named like its folder (`Rust/Rust.md`) sits in the middle and its children fill rings around it. Grown islands spread wider; young ones stay compact.
- **Moons.** Notes inside a `memories/` folder orbit their parent folder as small pebbles, so logs and AI memories stay visible without crowding the garden. Repeated titles (`Session summary - 12`, `- 13`…) collapse into the newest one.
- **Areas.** Each note directly inside an `Areas/` folder becomes a planet of its own.
- **Links** show as stepping stones when you hover a plant.

## Commands

- Open garden
- Go to a note in the garden
- Show the active note in the garden
- Watch the garden grow (timelapse by creation date)

## Frontmatter

| Key | Effect |
| --- | --- |
| `island: Name` | Send the note to that island |
| `garden: false` | Hide the note from the garden |
| `stage: tree` | Force a growth stage (`seed`, `sprout`, `plant`, `shrub`, `tree`) |
| `plant: pine` | Pick the tree style (`round`, `pine`, `bonsai`) |

## Settings

Group by, excluded folders (a name like `memories/` matches at any depth), island size and spacing, moon orbit, roughness of the island outlines, sand furrows, decorations, growth speed, labels, motion and colors.

## Install

Community plugins → search **Garden Zen**. Before it is listed, install it with [BRAT](https://github.com/TfTHacker/obsidian42-brat) using `racconrad/garden-zen`, or copy `main.js`, `manifest.json` and `styles.css` from the latest release into `.obsidian/plugins/garden-zen/`.

## License

MIT
