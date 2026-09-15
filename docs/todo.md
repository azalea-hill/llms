# Shared backlog

## Model research

Benchmark viable model sets for:

- **M5 Max, 128 GB**
- **M5 Ultra, 256 GB**
- **M5 Ultra, 512 GB**

The current Claude set is tuned for smaller machines. Determine which model combinations make sense at higher memory ceilings for both Claude Code and Codex. Consider dense, mixture-of-experts, and MLX-quantized variants. Target a 200K context per primary model and a cache-friendly loading strategy.

## Shared model configuration

Model identifiers are currently embedded in multiple scripts and documents. Now that both launchers exist, evaluate a small shared configuration source that does not couple their provider-specific settings.

## LM Studio settings helper

The setup documentation recommends adjusting LM Studio settings such as the
default context length and JIT-model unloading. Consider a safe helper that
shows and validates the proposed changes before updating user-level settings.

## Model version pinning

Model names are strings rather than immutable revisions. Investigate pinning
downloaded models to revision hashes so an upstream replacement cannot silently
change a working setup.

## Restore-to-idle command

There is no convenience command that unloads the model set and returns LM
Studio to an idle state. Add one if both setups would benefit from it.
