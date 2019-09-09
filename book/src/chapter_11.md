# Loading and Saving the Game

---

***About this tutorial***

*This tutorial is free and open source, and all code uses the MIT license - so you are free to do with it as you like. My hope is that you will enjoy the tutorial, and make great games!*

*If you enjoy this and would like me to keep writing, please consider supporting [my Patreon](https://www.patreon.com/blackfuture).*

---

In the last few chapters, we've focused on getting a playable (if not massively fun) game going. You can run around, slay monsters, and make use of various items. That's a great start! Most games let you stop playing, and come back later to continue. Fortunately, Rust (and associated libraries) makes it relatively easy.

# A Main Menu

If you're going to resume a game, you need somewhere from which to do so! A main menu also gives you the option to abandon your last save, possibly view credits, and generally tell the world that your game is here - and written by you. It's an important thing to have, so we'll put one together.

Being in the menu is a *state* - so we'll add it to the ever-expanding `RunState` enum. We want to include menu state inside it, so the definition winds up looking like this:

```rust
#[derive(PartialEq, Copy, Clone)]
pub enum RunState { AwaitingInput, 
    PreRun, 
    PlayerTurn, 
    MonsterTurn, 
    ShowInventory, 
    ShowDropItem, 
    ShowTargeting { range : i32, item : Entity},
    MainMenu { menu_selection : gui::MainMenuSelection }
}
```

In `gui.rs`, we add a couple of enum types to handle main menu selections:

```rust
#[derive(PartialEq, Copy, Clone)]
pub enum MainMenuSelection { NewGame, LoadGame, Quit }

#[derive(PartialEq, Copy, Clone)]
pub enum MainMenuResult { NoSelection{ selected : MainMenuSelection }, Selected{ selected: MainMenuSelection } }
```

Your GUI is probably now telling you that `main.rs` has errors! It's right - we need to handle the new `RunState` option. We'll need to change things around a bit to ensure that we aren't also rendering the GUI and map when in the menu. So we rearrange `tick`:

```rust
fn tick(&mut self, ctx : &mut Rltk) {
    let mut newrunstate;
    {
        let runstate = self.ecs.fetch::<RunState>();
        newrunstate = *runstate;
    }

    ctx.cls();        

    match newrunstate {
        RunState::MainMenu{..} => {}
        _ => {
            draw_map(&self.ecs, ctx);

            {
                let positions = self.ecs.read_storage::<Position>();
                let renderables = self.ecs.read_storage::<Renderable>();
                let map = self.ecs.fetch::<Map>();

                let mut data = (&positions, &renderables).join().collect::<Vec<_>>();
                data.sort_by(|&a, &b| b.1.render_order.cmp(&a.1.render_order) );
                for (pos, render) in data.iter() {
                    let idx = map.xy_idx(pos.x, pos.y);
                    if map.visible_tiles[idx] { ctx.set(pos.x, pos.y, render.fg, render.bg, render.glyph) }
                }

                gui::draw_ui(&self.ecs, ctx);
            } 
        }
    }
    ...
```

We'll also handle the `MainMenu` state in our large `match` for `RunState`:

```rust
RunState::MainMenu{ .. } => {
    let result = gui::main_menu(self, ctx);
    match result {
        gui::MainMenuResult::NoSelection{ selected } => newrunstate = RunState::MainMenu{ menu_selection: selected },
        gui::MainMenuResult::Selected{ selected } => {
            match selected {
                gui::MainMenuSelection::NewGame => newrunstate = RunState::PreRun,
                gui::MainMenuSelection::LoadGame => newrunstate = RunState::PreRun,
                gui::MainMenuSelection::Quit => { ::std::process::exit(0); }
            }
        }
    }
}
```

We're basically updating the state with the new menu selection, and if something has been selected we change the game state. For `Quit`, we simply terminate the process. For now, we'll make loading/starting a game do the same thing: go into the `PreRun` state to setup the game.

The last thing to do is to write the menu itself. In `menu.rs`:

```rust
pub fn main_menu(gs : &mut State, ctx : &mut Rltk) -> MainMenuResult {
    let runstate = gs.ecs.fetch::<RunState>();

    ctx.print_color_centered(15, RGB::named(rltk::YELLOW), RGB::named(rltk::BLACK), "Rust Roguelike Tutorial");
    
    if let RunState::MainMenu{ menu_selection : selection } = *runstate {
        if selection == MainMenuSelection::NewGame {
            ctx.print_color_centered(24, RGB::named(rltk::MAGENTA), RGB::named(rltk::BLACK), "Begin New Game");
        } else {
            ctx.print_color_centered(24, RGB::named(rltk::WHITE), RGB::named(rltk::BLACK), "Begin New Game");
        }

        if selection == MainMenuSelection::LoadGame {
            ctx.print_color_centered(25, RGB::named(rltk::MAGENTA), RGB::named(rltk::BLACK), "Load Game");
        } else {
            ctx.print_color_centered(25, RGB::named(rltk::WHITE), RGB::named(rltk::BLACK), "Load Game");
        }

        if selection == MainMenuSelection::Quit {
            ctx.print_color_centered(26, RGB::named(rltk::MAGENTA), RGB::named(rltk::BLACK), "Quit");
        } else {
            ctx.print_color_centered(26, RGB::named(rltk::WHITE), RGB::named(rltk::BLACK), "Quit");
        }

        match ctx.key {
            None => return MainMenuResult::NoSelection{ selected: selection },
            Some(key) => {
                match key {
                    VirtualKeyCode::Escape => { return MainMenuResult::NoSelection{ selected: MainMenuSelection::Quit } }
                    VirtualKeyCode::Up => {
                        let newselection;
                        match selection {
                            MainMenuSelection::NewGame => newselection = MainMenuSelection::Quit,
                            MainMenuSelection::LoadGame => newselection = MainMenuSelection::NewGame,
                            MainMenuSelection::Quit => newselection = MainMenuSelection::LoadGame
                        }
                        return MainMenuResult::NoSelection{ selected: newselection }
                    }
                    VirtualKeyCode::Down => {
                        let newselection;
                        match selection {
                            MainMenuSelection::NewGame => newselection = MainMenuSelection::LoadGame,
                            MainMenuSelection::LoadGame => newselection = MainMenuSelection::Quit,
                            MainMenuSelection::Quit => newselection = MainMenuSelection::NewGame
                        }
                        return MainMenuResult::NoSelection{ selected: newselection }
                    }
                    VirtualKeyCode::Return => return MainMenuResult::Selected{ selected : selection },
                    _ => return MainMenuResult::NoSelection{ selected: selection }
                }
            }
        }
    }

    MainMenuResult::NoSelection { selected: MainMenuSelection::NewGame }
}
```

That's a bit of a mouthful, but it displays menu options and lets you select them with the up/down keys and enter. It's very careful to not modify state itself, to keep things clear.

# Including Serde

`Serde` is pretty much the gold-standard for serialization in Rust. It makes a lot of things easier! So the first step is to include it. In your project's `Cargo.toml` file, we'll expand the `dependencies` section to include it:

```toml
[dependencies]
rltk = { git = "https://github.com/thebracket/rltk_rs", features = ["serialization"] }
specs = { version = "0.15.0", features = ["serde"] }
specs-derive = "0.4.0"
serde= { version = "1.0.93", features = ["derive"] }
serde_json = "1.0.39"
```

It may be worth calling `cargo run` now - it will take a while, downloading the new dependencies (and all of their dependencies) and building them for you. It should keep them around so you don't have to wait this long every time you build.

# Adding a "SaveGame" state

We'll extend `RunState` once more to support game saving:

```rust
#[derive(PartialEq, Copy, Clone)]
pub enum RunState { AwaitingInput, 
    PreRun, 
    PlayerTurn, 
    MonsterTurn, 
    ShowInventory, 
    ShowDropItem, 
    ShowTargeting { range : i32, item : Entity},
    MainMenu { menu_selection : gui::MainMenuSelection },
    SaveGame
}
```

In `tick`, we'll add dummy code for now:

```rust
RunState::SaveGame => {
    newrunstate = RunState::MainMenu{ menu_selection : gui::MainMenuSelection::LoadGame };
}
```

In `player.rs`, we'll add another keyboard handler - escape:

```rust
// Save and Quit
VirtualKeyCode::Escape => return RunState::SaveGame,
```

If you `cargo run` now, you can start a game and press escape to quit to the menu.

# Getting started with saving the game

Now that the scaffolding is in place, it's time to actually save something! Lets start simple, to get a feel for Serde. In the `tick` function, we extend the save system to just dump a JSON representation of the map to the console:

```rust
RunState::SaveGame => {
    let data = serde_json::to_string(&*self.ecs.fetch::<Map>()).unwrap();
    println!("{}", data);

    newrunstate = RunState::MainMenu{ menu_selection : gui::MainMenuSelection::LoadGame };
}
```

We'll also need to add an `extern crate serde;` to the top of `main.rs`.

This won't compile, because we need to tell `Map` to serialize itself! Fortunately, `serde` provides some helpers to make this easy. At the top of `map.rs`, we add `use serde::{Serialize, Deserialize};`. We then decorate the map to derive serialization and de-serialization code:

```rust
#[derive(Default, Serialize, Deserialize)]
pub struct Map {
    pub tiles : Vec<TileType>,
    pub rooms : Vec<Rect>,
    pub width : i32,
    pub height : i32,
    pub revealed_tiles : Vec<bool>,
    pub visible_tiles : Vec<bool>,
    pub blocked : Vec<bool>,

    #[serde(skip_serializing)]
    #[serde(skip_deserializing)]
    pub tile_content : Vec<Vec<Entity>>
}
```

Note that we've decorated `tile_content` with directives to not serialize/de-serialize it. This prevents us from needing to store the entities, and since this data is rebuilt every frame - it doesn't matter. The game still won't compile; we need to add similar decorators to `TileType` and `Rect`:

```rust
#[derive(PartialEq, Copy, Clone, Serialize, Deserialize)]
pub enum TileType {
    Wall, Floor
}
```

```rust
#[derive(PartialEq, Copy, Clone, Serialize, Deserialize)]
pub struct Rect {
    pub x1 : i32,
    pub x2 : i32,
    pub y1 : i32,
    pub y2 : i32
}
```

Lastly, we should extend the game saving code to dump the map to the console:

```rust
let data = serde_json::to_string(&*self.ecs.fetch::<Map>()).unwrap();
println!("{}", data);
```

If you `cargo run` the project now, when you hit escape it will dump a huge blob of JSON data to the console. That's the game map!

# Saving entity state

Now that we've seen how useful `serde` is, we should start to use it for the game itself. This is harder than one might expect, because of how `specs` handles `Entity` structures: their ID # is purely synthetic, with no guaranty that you'll get the same one next time! Also, you may not want to save *everything* - so `specs` introduces a concept of *markers* to help with this. It winds up being a bit more of a mouthful than it really needs to be, but gives a pretty powerful serialization system.

## Introducing Markers

First of all, in `main.rs` we'll tell Rust that we'd like to make use of the marker functionality:

```rust
use specs::saveload::{SimpleMarker, SimpleMarkerAllocator};
```

In `components.rs`, we'll add a marker type:

```rust
pub struct SerializeMe;
```

Back in `main.rs`, we'll add `SerializeMe` to the list of things that we *register*:

```rust
gs.ecs.register::<SimpleMarker<SerializeMe>>();
```

We'll also add an entry to the ECS resources, which gets used to determine the next identity:

```rust
gs.ecs.insert(SimpleMarkerAllocator::<SerializeMe>::new());
```

Finally, in `spawners.rs` we tell each entity builder to include the marker. Here's the complete entry for the `Player`:

```rust
pub fn player(ecs : &mut World, player_x : i32, player_y : i32) -> Entity {
    ecs
        .create_entity()
        .with(Position { x: player_x, y: player_y })
        .with(Renderable {
            glyph: rltk::to_cp437('@'),
            fg: RGB::named(rltk::YELLOW),
            bg: RGB::named(rltk::BLACK),
            render_order: 0
        })
        .with(Player{})
        .with(Viewshed{ visible_tiles : Vec::new(), range: 8, dirty: true })
        .with(Name{name: "Player".to_string() })
        .with(CombatStats{ max_hp: 30, hp: 30, defense: 2, power: 5 })
        .marked::<SimpleMarker<SerializeMe>>()
        .build()
}
```

The new line (`.marked::<SimpleMarker<SerializeMe>>()`) needs to be repeated for all of our spawners in this file.

## Actually Serializing Something



**The source code for this chapter may be found [here](https://github.com/thebracket/rustrogueliketutorial/tree/master/chapter-10-ranged)**

---

Copyright (C) 2019, Herbert Wolverson.

---