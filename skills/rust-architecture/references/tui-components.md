# TUI Component Architecture

Component-based structure for Ratatui terminal applications.
Each component owns its state, rendering, and input handling.
The App coordinates components through a shared action system.

Based on the Ratatui component template.

---

## Project Structure

```
my-tui-app/
├── build.rs
├── Cargo.toml
└── src/
    ├── main.rs            # Entry point, CLI parsing, logging init
    ├── app.rs             # App struct, main loop coordination
    ├── tui.rs             # Terminal setup/teardown, raw mode
    ├── action.rs          # Action enum (Tick, Render, Quit, custom)
    ├── cli.rs             # clap argument definitions
    ├── config.rs          # Keybindings, settings
    ├── errors.rs          # Application error type
    ├── logging.rs         # tracing/log setup
    ├── components.rs      # Component trait + re-exports
    └── components/
        ├── fps.rs         # FPS counter component
        └── home.rs        # Main home screen component
```

---

## File Responsibilities

### main.rs

Thin entry point. Parses CLI args, initializes logging, creates
the App, and runs it. No application logic here.

```rust
fn main() -> Result<()> {
    let args = cli::Args::parse();
    logging::init(args.log_level)?;
    let mut app = App::new(args)?;
    app.run()
}
```

### app.rs

The central coordinator. Owns all components, runs the main loop,
and dispatches actions. This is where the event-action-update-draw
cycle lives.

```rust
pub struct App {
    should_quit: bool,
    tick_rate: f64,
    frame_rate: f64,
    components: Vec<Box<dyn Component>>,
}

impl App {
    pub fn new(args: Args) -> Result<Self> {
        let home = Home::new();
        let fps = FpsCounter::new();
        Ok(Self {
            should_quit: false,
            tick_rate: args.tick_rate,
            frame_rate: args.frame_rate,
            components: vec![Box::new(home), Box::new(fps)],
        })
    }
}
```

### tui.rs

Terminal lifecycle management. Enters raw mode and alternate
screen on start, restores on exit. Provides the event stream
(keyboard, mouse, resize) and the rendering frame.

```rust
pub struct Tui {
    terminal: Terminal<CrosstermBackend<Stderr>>,
    event_rx: UnboundedReceiver<Event>,
    event_tx: UnboundedSender<Event>,
}

impl Tui {
    pub fn new() -> Result<Self> { /* setup terminal */ }
    pub fn enter(&mut self) -> Result<()> { /* enable raw mode */ }
    pub fn exit(&mut self) -> Result<()> { /* restore terminal */ }
    pub fn draw(&mut self, f: impl FnOnce(&mut Frame)) -> Result<()> { /* render */ }
}
```

### action.rs

All possible actions in the application. Components emit actions
in response to events, and consume actions during update.

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum Action {
    Tick,
    Render,
    Quit,
    Resize(u16, u16),
    // Custom actions per feature
    NavigateUp,
    NavigateDown,
    Select,
}
```

### cli.rs

Command-line argument definitions using clap.

### config.rs

Keybinding mappings and application settings. Maps key events
to actions so keybindings are configurable without changing
component code.

### errors.rs

Application-level error type. Typically uses `thiserror` or
`color_eyre` for rich error reporting in the terminal.

### logging.rs

Sets up `tracing` to log to a file (not stdout/stderr, since
the terminal UI owns those). Useful for debugging without
disrupting the TUI.

### components.rs

Defines the Component trait and re-exports all components:

```rust
pub mod fps;
pub mod home;

pub use fps::FpsCounter;
pub use home::Home;
```

---

## Component Trait

The contract every component implements:

```rust
pub trait Component {
    fn init(&mut self) -> Result<()> {
        Ok(())
    }

    fn handle_events(
        &mut self,
        event: Option<Event>,
    ) -> Result<Option<Action>> {
        Ok(None)
    }

    fn update(
        &mut self,
        action: Action,
    ) -> Result<Option<Action>> {
        Ok(None)
    }

    fn draw(
        &mut self,
        frame: &mut Frame,
        area: Rect,
    ) -> Result<()>;
}
```

- **init** — One-time setup when the component is first created.
- **handle_events** — Converts raw terminal events (key press,
  mouse click) into Actions. Returns `None` if the event is
  not relevant to this component.
- **update** — Reacts to Actions by modifying internal state.
  Can return a new Action to chain effects.
- **draw** — Renders the component into the given area of the
  frame. Called every render cycle.

---

## App Loop Pattern

The main loop follows: poll events → dispatch to components →
collect actions → update components → draw.

```rust
impl App {
    pub fn run(&mut self) -> Result<()> {
        let mut tui = Tui::new()?;
        tui.enter()?;

        for component in self.components.iter_mut() {
            component.init()?;
        }

        loop {
            // 1. Poll for terminal events
            let event = tui.next_event()?;

            // 2. Let each component handle the event
            let mut actions = Vec::new();
            for component in self.components.iter_mut() {
                if let Some(action) = component.handle_events(event.clone())? {
                    actions.push(action);
                }
            }

            // 3. Process actions through all components
            for action in actions {
                if action == Action::Quit {
                    self.should_quit = true;
                }
                for component in self.components.iter_mut() {
                    component.update(action.clone())?;
                }
            }

            // 4. Draw all components
            tui.draw(|frame| {
                for component in self.components.iter_mut() {
                    component.draw(frame, frame.area()).ok();
                }
            })?;

            if self.should_quit {
                break;
            }
        }

        tui.exit()?;
        Ok(())
    }
}
```

---

## TUI with Vertical Slices

For TUI apps with real business logic — data access, services,
domain models — combine component-based UI with vertical slices.
Each feature slice owns its components alongside its service layer.

```
my-tui-app/
├── Cargo.toml
└── src/
    ├── main.rs
    ├── app.rs                     # Coordinates features + shared components
    ├── tui.rs
    ├── action.rs
    ├── cli.rs
    ├── config.rs
    ├── errors.rs
    ├── logging.rs
    ├── components.rs              # Component trait definition
    ├── components/                # Shared/generic components
    │   ├── fps.rs
    │   └── status_bar.rs
    ├── features/
    │   ├── mod.rs
    │   ├── users/
    │   │   ├── mod.rs
    │   │   ├── components/        # Feature-specific components
    │   │   │   ├── user_list.rs
    │   │   │   └── user_detail.rs
    │   │   ├── service.rs
    │   │   ├── model.rs
    │   │   └── repository.rs
    │   └── orders/
    │       ├── mod.rs
    │       ├── components/
    │       │   └── order_table.rs
    │       ├── service.rs
    │       ├── model.rs
    │       └── repository.rs
    ├── shared/
    │   ├── mod.rs
    │   ├── db.rs
    │   ├── errors.rs
    │   └── config.rs
    └── domain/
        └── value_objects.rs
```

**How the layers connect:** Feature components receive a reference
to their slice's service. The component handles UI concerns (rendering,
input), the service handles business logic and data access.

```rust
// features/users/components/user_list.rs

pub struct UserList {
    users: Vec<User>,
    selected: usize,
    svc: Arc<UserService>,
}

impl UserList {
    pub fn new(svc: Arc<UserService>) -> Self {
        Self { users: Vec::new(), selected: 0, svc }
    }
}

impl Component for UserList {
    fn init(&mut self) -> Result<()> {
        self.users = self.svc.list_all()?;
        Ok(())
    }

    fn update(&mut self, action: Action) -> Result<Option<Action>> {
        match action {
            Action::NavigateUp => {
                self.selected = self.selected.saturating_sub(1);
            }
            Action::NavigateDown => {
                if self.selected < self.users.len().saturating_sub(1) {
                    self.selected += 1;
                }
            }
            Action::Refresh => {
                self.users = self.svc.list_all()?;
            }
            _ => {}
        }
        Ok(None)
    }

    fn draw(&mut self, frame: &mut Frame, area: Rect) -> Result<()> {
        let items: Vec<ListItem> = self.users.iter()
            .map(|u| ListItem::new(u.name.as_str()))
            .collect();
        let list = List::new(items)
            .highlight_style(Style::default().reversed());
        let mut state = ListState::default()
            .with_selected(Some(self.selected));
        frame.render_stateful_widget(list, area, &mut state);
        Ok(())
    }
}
```

**Wiring in app.rs:** Build services first (like `router.rs` in
a web API), then inject into feature components:

```rust
// app.rs

impl App {
    pub fn new(args: Args) -> Result<Self> {
        let pool = shared::db::create_pool(&args.database_url)?;

        let user_repo = Arc::new(PgUserRepository::new(pool.clone()));
        let user_svc = Arc::new(UserService::new(user_repo));
        let user_list = UserList::new(user_svc);

        let fps = FpsCounter::new();
        let status = StatusBar::new();

        Ok(Self {
            should_quit: false,
            components: vec![
                Box::new(user_list),
                Box::new(fps),
                Box::new(status),
            ],
        })
    }
}
```

**Key differences from plain component-based:**

| Concern | Plain TUI | TUI + Vertical Slices |
|---------|-----------|----------------------|
| Components location | `src/components/` | Feature-specific in `features/X/components/`, shared in `src/components/` |
| Business logic | Inside components or app.rs | In `features/X/service.rs` |
| Data access | Direct in components | Via `features/X/repository.rs` trait |
| Adding a feature | New component file | New feature directory with components + service + model |
| Cross-feature data | Pass through App | Cross-slice strategies (API traits, shared types) |

**When to use this hybrid:** When the TUI app has data persistence,
multiple features with distinct domain logic, or when you want the
same service layer to be reusable across a TUI and a future web API.
For simple single-screen TUI tools, the plain component structure
is sufficient.

---

## Adding a New Component

1. Create `src/components/my_widget.rs`
2. Implement the `Component` trait (at minimum: `draw`)
3. Add `pub mod my_widget;` to `src/components.rs`
4. Instantiate in `App::new()` and add to `self.components`

If the component needs custom actions, add variants to the
`Action` enum in `action.rs`.

---

## Common Dependencies

```toml
[dependencies]
ratatui = "0.29"
crossterm = "0.28"
tokio = { version = "1", features = ["full"] }
clap = { version = "4", features = ["derive"] }
serde = { version = "1", features = ["derive"] }
tracing = "0.1"
tracing-subscriber = "0.3"
color-eyre = "0.6"

[build-dependencies]
anyhow = "1"
vergen-gix = { version = "1", features = ["build", "cargo"] }
```
