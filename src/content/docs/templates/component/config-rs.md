---
title: config.rs
---

At the moment, our keys are hard coded into the app.

```rust {filename="components/home.rs"}
impl Component for Home {

  fn handle_key_events(&mut self, key: KeyEvent) -> Action {
    match self.mode {
      Mode::Normal | Mode::Processing => {
        match key.code {
          KeyCode::Char('q') => Action::Quit,
          KeyCode::Char('d') if key.modifiers.contains(KeyModifiers::CONTROL) => Action::Quit,
          KeyCode::Char('c') if key.modifiers.contains(KeyModifiers::CONTROL) => Action::Quit,
          KeyCode::Char('z') if key.modifiers.contains(KeyModifiers::CONTROL) => Action::Suspend,
          KeyCode::Char('?') => Action::ToggleShowHelp,
          KeyCode::Char('j') => Action::ScheduleIncrement,
          KeyCode::Char('k') => Action::ScheduleDecrement,
          KeyCode::Char('/') => Action::EnterInsert,
          _ => Action::Tick,
        }
      },
      Mode::Insert => {
        match key.code {
          KeyCode::Esc => Action::EnterNormal,
          KeyCode::Enter => Action::EnterNormal,
          _ => {
            self.input.handle_event(&crossterm::event::Event::Key(key));
            Action::Update
          },
        }
      },
    }
  }
```

If a user wants to press `Up` and `Down` arrow key to `ScheduleIncrement` and `ScheduleDecrement`,
the only way for them to do it is having to make changes to the source code and recompile the app.
It would be better to provide a way for users to set up a configuration file that maps key presses
to actions.

For example, assume we want a user to be able to set up a keyevents-to-actions mapping in a
`config.toml` file like below:

```toml
[keymap]
"q" = "Quit"
"j" = "ScheduleIncrement"
"k" = "ScheduleDecrement"
"l" = "ToggleShowHelp"
"/" = "EnterInsert"
"ESC" = "EnterNormal"
"Enter" = "EnterNormal"
"Ctrl-d" = "Quit"
"Ctrl-c" = "Quit"
"Ctrl-z" = "Suspend"
```

We can set up a `Config` struct using
[the excellent `config` crate](https://docs.rs/config/0.13.3/config/):

```rust
use std::collections::HashMap;

use color_eyre::eyre::Result;
use crossterm::event::KeyEvent;
use serde_derive::Deserialize;

use crate::action::Action;

#[derive(Clone, Debug, Default, Deserialize)]
pub struct Config {
  #[serde(default)]
  pub keymap: KeyMap,
}

#[derive(Clone, Debug, Default, Deserialize)]
pub struct KeyMap(pub HashMap<KeyEvent, Action>);

impl Config {
  pub fn new() -> Result<Self, config::ConfigError> {
    let mut builder = config::Config::builder();
    builder = builder
      .add_source(config::File::from(config_dir.join("config.toml")).format(config::FileFormat::Toml).required(false));
    builder.build()?.try_deserialize()
  }
}
```

We are using `serde` to deserialize from a TOML file.

Now the default `KeyEvent` serialized format is not very user friendly, so let's implement our own
version:

```rust
#[derive(Clone, Debug, Default)]
pub struct KeyMap(pub HashMap<KeyEvent, Action>);

impl<'de> Deserialize<'de> for KeyMap {
  fn deserialize<D>(deserializer: D) -> Result<Self, D::Error> where D: Deserializer<'de>,
  {
    struct KeyMapVisitor;
    impl<'de> Visitor<'de> for KeyMapVisitor {
      type Value = KeyMap;
      fn visit_map<M>(self, mut access: M) -> Result<KeyMap, M::Error>
      where
        M: MapAccess<'de>,
      {
        let mut keymap = HashMap::new();
        while let Some((key_str, action)) = access.next_entry::<String, Action>()? {
          let key_event = parse_key_event(&key_str).map_err(de::Error::custom)?;
          keymap.insert(key_event, action);
        }
        Ok(KeyMap(keymap))
      }
    }
    deserializer.deserialize_map(KeyMapVisitor)
  }
}
```

Now all we need to do is implement a `parse_key_event` function.
[You can check the source code for an example of this implementation](https://github.com/ratatui-org/templates/blob/main/async/template/src/config.rs#L105-L109).

With that implementation complete, we can add a `HashMap` to store a map of `KeyEvent`s and `Action`
in the `Home` component:

```rust {filename="components/home.rs"}
#[derive(Default)]
pub struct Home {
  ...
  pub keymap: HashMap<KeyEvent, Action>,
}
```

Now we have to create an instance of `Config` and pass the keymap to `Home`:

```rust
impl App {
  pub fn new(tick_rate: (u64, u64)) -> Result<Self> {
    let h = Home::new();
    let config = Config::new()?;
    let h = h.keymap(config.keymap.0.clone());
    let home = Arc::new(Mutex::new(h));
    Ok(Self { tick_rate, home, should_quit: false, should_suspend: false, config })
  }
}
```

:::tip

You can create different keyevent presses to map to different actions based on the mode of the app
by adding more sections into the toml configuration file.

:::

And in the `handle_key_events` we get the `Action` that should to be performed from the `HashMap`
directly.

```rust
impl Component for Home {
  fn handle_key_events(&mut self, key: KeyEvent) -> Action {
    match self.mode {
      Mode::Normal | Mode::Processing => {
        if let Some(action) = self.keymap.get(&key) {
          *action
        } else {
          Action::Tick
        }
      },
      Mode::Insert => {
        match key.code {
          KeyCode::Esc => Action::EnterNormal,
          KeyCode::Enter => Action::EnterNormal,
          _ => {
            self.input.handle_event(&crossterm::event::Event::Key(key));
            Action::Update
          },
        }
      },
    }
  }
}
```

In the template, it is set up to handle `Vec<KeyEvent>` mapped to an `Action`. This allows you to
map for example:

- `<g><j>` to `Action::GotoBottom`
- `<g><k>` to `Action::GotoTop`

:::note

Remember, if you add a new `Action` variant you also have to update the `deserialize` method
accordingly.

:::

And because we are now using multiple keys as input, you have to update the `app.rs` main loop
accordingly to handle that:

```rust
    // -- snip --
    loop {
      if let Some(e) = tui.next().await {
        match e {
          // -- snip --
          tui::Event::Key(key) => {
            if let Some(keymap) = self.config.keybindings.get(&self.mode) {
              // If the key is a single key action
              if let Some(action) = keymap.get(&vec![key.clone()]) {
                log::info!("Got action: {action:?}");
                action_tx.send(action.clone())?;
              } else {
                // If the key was not handled as a single key action,
                // then consider it for multi-key combinations.
                self.last_tick_key_events.push(key);

                // Check for multi-key combinations
                if let Some(action) = keymap.get(&self.last_tick_key_events) {
                  log::info!("Got action: {action:?}");
                  action_tx.send(action.clone())?;
                }
              }
            };
          },
          _ => {},
        }
        // -- snip --
      }
      while let Ok(action) = action_rx.try_recv() {
        // -- snip --
        for component in self.components.iter_mut() {
          if let Some(action) = component.update(action.clone())? {
            action_tx.send(action)?
          };
        }
      }
      // -- snip --
    }
    // -- snip --
```

Here's the JSON configuration we use for the counter application:

```json
{{#include @code/async-template-counter/.config/config.json5}}
```
