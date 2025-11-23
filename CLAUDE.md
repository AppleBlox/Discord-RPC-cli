# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A Discord Rich Presence (RPC) CLI client written in Rust. This application allows users to set custom Discord Rich Presence status via command-line arguments, with support for images, buttons, timestamps, party info, an AFK mode that automatically detects user inactivity, and an update mode that accepts JSON from stdin for dynamic RPC updates without reconnecting.

## Build and Development Commands

### Building
```bash
# Standard debug build
cargo build

# Release build
cargo build --release

# Cross-compile for macOS targets (requires target installed)
rustup target add x86_64-apple-darwin
rustup target add aarch64-apple-darwin
cargo build --release --target x86_64-apple-darwin
cargo build --release --target aarch64-apple-darwin

# Create universal macOS binary (macOS only)
lipo -create \
  ./target/x86_64-apple-darwin/release/discord-rpc-cli \
  ./target/aarch64-apple-darwin/release/discord-rpc-cli \
  -output ./target/release/discord-rpc-cli
```

### Testing
```bash
# Run all tests
cargo test

# Run tests verbosely
cargo test --verbose
```

### Running
```bash
# Run with cargo
cargo run -- [OPTIONS]

# Run built binary
./target/release/discord-rpc-cli [OPTIONS]

# Example: Set custom RPC (one-time)
./target/release/discord-rpc-cli -c YOUR_CLIENT_ID -d "Playing a game" -s "In menu"

# Example: Update mode (dynamic updates via stdin)
./target/release/discord-rpc-cli -c YOUR_CLIENT_ID --update
# Then send JSON objects like:
# {"state": "In game", "details": "Level 5"}
# {"state": "In menu", "details": "Browsing items", "large_image": "menu_icon"}
```

## Code Architecture

### Entry Point (src/main.rs)
The main logic is in `src/main.rs` and follows this flow:

1. **Argument Parsing**: Uses clap to parse CLI arguments defined in `src/cli.rs`
2. **Three Main Modes**:
   - **Normal RPC Mode** (default): Sets custom Discord RPC based on user-provided arguments, runs once or indefinitely
   - **Update Mode** (`--update` flag): Maintains a persistent connection and reads JSON from stdin to dynamically update RPC without reconnecting
   - **AFK Mode** (`--afk` flag): Automatically detects user idle time and displays AFK status

3. **Activity Building Pattern**: The code uses a builder pattern to construct Discord activities incrementally. Each optional parameter conditionally adds to the activity:
   ```
   activity → activity_state → activity_details → activity_with_assets →
   activity_button_1 → activity_button_2 → activity_time → ... → final activity
   ```
   This allows for flexible combinations of RPC features.

4. **Validation and Exit Conditions**: Multiple validation checks at src/main.rs:54-81 ensure:
   - Client ID is provided (unless AFK mode)
   - Buttons are numbered correctly (button 1 must exist before button 2)
   - Timestamps are mutually exclusive with enable_time
   - Party ID requires party_size
   - Match secrets require match_id

### CLI Structure (src/cli.rs)
Uses clap's derive API with a single `Cli` struct containing all command-line options. Arguments use `default_value="__None"` as a sentinel for optional string parameters (instead of `Option<String>`), which simplifies the conditional logic in main.

### Key Dependencies
- `discord-rich-presence`: Core library for Discord IPC communication
- `clap`: CLI argument parsing with derive macros
- `colored`: Terminal color output (can be disabled with `--disable_color`)
- `user-idle`: System idle time detection for AFK mode
- `serde` / `serde_json`: JSON deserialization for update mode

### Special Values and Conventions
- `"__None"`: Sentinel value indicating an optional parameter wasn't provided
- `-1`: Sentinel for unset numeric values (timestamps, exit_after)
- Party size format: `"[current,max]"` (e.g., `"[2,10]"`)
- Helper functions `check_current_party_size()` and `check_max_party_size()` parse party size strings

### Update Mode Implementation
When `--update` flag is enabled (src/main.rs:136-206):
1. Requires client ID to be provided via `--clientid`
2. Establishes a single persistent connection to Discord IPC
3. Reads newline-delimited JSON from stdin in a loop
4. Each JSON object is deserialized into the `RpcUpdate` struct (all fields optional)
5. Uses `build_activity_from_update()` helper to construct activity from JSON
6. Updates activity via the existing IPC connection (no reconnection overhead)
7. Continues until stdin closes or Ctrl+C is pressed

**JSON Format**: All fields are optional. Example:
```json
{"state": "In game", "details": "Level 5", "large_image": "game_icon"}
{"party_size": [2, 4], "party_id": "my-party"}
{"enable_time": true, "button_text_1": "Join", "button_url_1": "https://example.com"}
```

### AFK Mode Implementation
When `--afk` flag is enabled:
1. Uses hardcoded client ID `871404899915665438`
2. Polls system idle time every `afk_update` seconds (default: 20s)
3. Sets activity after user is idle for `afk_after` minutes (default: 5min)
4. Displays idle duration in activity details
5. Clears activity when user becomes active again

## Important Notes

- The application runs indefinitely by default (infinite loop at src/main.rs:249). Use `--exit_after` or Ctrl+C to exit.
- Button ordering is strict: button 2 cannot be set without button 1 being set first (validated at src/main.rs:64-68).
- Timestamps have special rules: `--enable_time` (current time) cannot be used with explicit `--start_time` or `--end_time` (validated at src/main.rs:69-72).
- Color output is enabled by default throughout the application. Use `--disable_color` to disable it.
- The codebase uses Rust 2024 edition (Cargo.toml:4).
