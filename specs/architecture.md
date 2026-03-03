# Nix-based Container Development Environment — Architecture Proposal

## Status

Draft — starting point for discussion and implementation.

## Context

This project replaces a Docker multi-stage build system
([jorgengundersen/devenv](https://github.com/jorgengundersen/devenv)) with a
Nix-based approach. The original system uses one Dockerfile per tool, multi-stage
`COPY --from` aggregation, and a three-layer image hierarchy
(repo-base → devenv-base → devenv → project). The Nix port preserves the core
ideas — composable tool sets, persistent container model, XDG volumes — while
gaining reproducibility, a single package source (~100k packages in nixpkgs),
and declarative profile composition.

The original `devenv` and `build-devenv` scripts are **not** carried over. A
separate CLI tool (potentially `jorgengundersen/devenv-cli`) will serve as the
human interface and point to this repo for environment definitions.

---

## Design Principles

1. **Declarative** — All environment definitions are Nix expressions. No
   imperative install scripts, no `curl | bash`, no manual GitHub release URL
   parsing.
2. **Composable** — Environments are assembled from profiles (package groups).
   Adding or removing a tool is a one-line change.
3. **Reproducible** — `flake.lock` pins the entire dependency tree.
   `nix flake update` bumps everything atomically.
4. **Single source of truth** — All packages come from nixpkgs. Stable channel
   by default, unstable overlay for fast-moving packages.
5. **Architecture-portable** — Target system is defined once; all Nix files
   derive it. Supports future migration to aarch64 (e.g., Apple Silicon).

---

## Directory Structure

```
devenv-nix/
├── flake.nix                    # Entry point — all outputs
├── flake.lock                   # Pinned inputs (auto-generated)
├── specs/
│   └── architecture.md          # This document
├── lib/
│   └── mkDevContainer.nix       # Helper: profiles → OCI image
├── profiles/
│   ├── common-utils.nix         # tree, less, man-db, file, zip, procps, etc.
│   ├── cloud.nix                # awscli, kubectl, terraform
│   ├── containers.nix           # docker-cli, docker-compose
│   ├── data.nix                 # jq, yq, dolt
│   ├── editors.nix              # neovim, tree-sitter CLI
│   ├── go.nix                   # go, gopls, delve
│   ├── node.nix                 # nodejs, pnpm/npm
│   ├── python.nix               # python3, uv
│   ├── rust.nix                 # rustc, cargo, rust-analyzer
│   ├── search.nix               # ripgrep, fd, fzf
│   ├── shell.nix                # bash, starship, zoxide
│   └── vcs.nix                  # git, gh, git-lfs
├── environments/
│   ├── general.nix              # General-purpose (most profiles)
│   ├── headless.nix             # Agentic/CI — no SSH, no editors
│   └── examples/
│       ├── python-project.nix   # Project-specific example
│       └── custom.nix           # Special-purpose example
└── docker/
    └── entrypoint.sh            # Container entrypoint
```

### What is intentionally absent

- **Home Manager modules** — Dotfiles are bind-mounted from the host at runtime
  (the current devenv pattern). Home Manager integration will be added later
  when host dotfiles are converted to Nix.
- **`devenv` / `build-devenv` scripts** — A separate CLI project will handle
  the human interface (container lifecycle, volume management, etc.).
- **Docker Compose** — Future consideration for sidecar services (e.g., dolt
  server). For now, dolt is part of the entrypoint.

---

## System Architecture Portability

The target system (`x86_64-linux`, `aarch64-linux`, etc.) is defined **once**
in `flake.nix` and threaded through to all other files. No profile or
environment file references a system string directly.

```nix
# flake.nix (excerpt)
let
  supportedSystems = [ "x86_64-linux" "aarch64-linux" ];
  forAllSystems = f: nixpkgs.lib.genAttrs supportedSystems (system:
    f nixpkgs.legacyPackages.${system}
  );
in {
  packages = forAllSystems (pkgs: {
    # pkgs is already system-specific — profiles just use it
    general = import ./environments/general.nix { inherit pkgs profiles mkDevContainer; };
  });
}
```

When the time comes to add macOS or ARM support, only the `supportedSystems`
list needs updating.

---

## Nixpkgs Channel Strategy: Stable + Unstable Overlay

Use a **stable** nixpkgs channel as the default for predictability. Fast-moving
packages (beads, dolt, opencode, copilot-cli, etc.) are pulled from
**nixpkgs-unstable** via an overlay.

```nix
# flake.nix (excerpt)
inputs = {
  nixpkgs.url = "github:NixOS/nixpkgs/nixos-24.11";           # stable
  nixpkgs-unstable.url = "github:NixOS/nixpkgs/nixpkgs-unstable"; # bleeding edge
};

outputs = { self, nixpkgs, nixpkgs-unstable, ... }: let
  # Overlay that pulls specific packages from unstable
  unstableOverlay = system: final: prev: let
    unstable = nixpkgs-unstable.legacyPackages.${system};
  in {
    # Fast-moving packages — always latest
    dolt = unstable.dolt;
    opencode = unstable.opencode;
    # Add more as needed
  };

  forAllSystems = f: nixpkgs.lib.genAttrs supportedSystems (system:
    f (import nixpkgs {
      inherit system;
      overlays = [ (unstableOverlay system) ];
    })
  );
in { /* ... */ };
```

Profiles simply reference `pkgs.dolt`, `pkgs.opencode`, etc. — the overlay
transparently provides the unstable version. This is a standard Nix pattern.

For packages **not in nixpkgs at all** (e.g., beads, copilot-cli), custom
derivations or fetchurl-based packages can be defined in a `packages/` directory
and added to the overlay.

---

## Core Abstraction: `mkDevContainer`

A single function that takes profiles and options, and produces an OCI image.
Uses `streamLayeredImage` to avoid writing the full tarball to the Nix store.

```nix
# lib/mkDevContainer.nix
{ pkgs, lib }:

{ name
, tag ? "latest"
, profiles ? []              # List of package lists
, extraPackages ? []         # Ad-hoc packages for project-specific needs
, enableSSH ? false          # Include SSH server
, user ? "devuser"
, maxLayers ? 120
, entrypoint ? null          # Optional entrypoint script
}:

let
  allPackages = builtins.concatLists profiles
    ++ extraPackages
    ++ lib.optionals enableSSH [ pkgs.openssh pkgs.tini ];

in pkgs.dockerTools.streamLayeredImage {
  inherit name tag maxLayers;
  contents = [
    pkgs.dockerTools.fakeNss
    pkgs.dockerTools.usrBinEnv
    pkgs.dockerTools.binSh
    pkgs.dockerTools.caCertificates
    (pkgs.buildEnv {
      name = "${name}-env";
      paths = allPackages;
      pathsToLink = [ "/bin" "/lib" "/share" "/etc" ];
    })
  ];

  fakeRootCommands = ''
    mkdir -p ./home/${user}/.local/{share,cache,state}
    mkdir -p ./home/${user}/.config
    mkdir -p ./tmp
    chmod 1777 ./tmp
  '';

  config = {
    Cmd = [ "${pkgs.bashInteractive}/bin/bash" ];
    User = user;
    WorkingDir = "/home/${user}";
    Env = [
      "HOME=/home/${user}"
      "USER=${user}"
      "LANG=C.UTF-8"
      "SSL_CERT_FILE=${pkgs.cacert}/etc/ssl/certs/ca-bundle.crt"
    ];
  } // lib.optionalAttrs (entrypoint != null) {
    Entrypoint = [ entrypoint ];
  };
}
```

---

## Profiles

Each profile is a function `{ pkgs } -> [ list of packages ]`. Profiles are
the unit of composition — they can be combined freely.

```nix
# profiles/search.nix
{ pkgs }: with pkgs; [
  ripgrep
  fd
  fzf
]
```

```nix
# profiles/python.nix
{ pkgs }: with pkgs; [
  python3
  uv
]
```

```nix
# profiles/common-utils.nix
{ pkgs }: with pkgs; [
  tree
  less
  man-db
  file
  unzip
  zip
  procps
  lsof
  iproute2
  iputils
  dnsutils
  netcat-gnu
]
```

Profile granularity is intentionally undecided. The current grouping is by
domain (search, vcs, editors, etc.). Through experimentation, profiles may be
split finer (one package per profile, mirroring the original Dockerfile-per-tool
pattern) or kept coarse. The `mkDevContainer` API supports either approach
without changes — profiles are just lists that get concatenated.

---

## Environments

Environments select and combine profiles into a complete image. Three types are
supported.

### General-Purpose

The default development environment. Includes most profiles. Equivalent to the
current `devenv` image.

```nix
# environments/general.nix
{ pkgs, profiles, mkDevContainer, ... }:

mkDevContainer {
  name = "devenv";
  enableSSH = true;
  profiles = with profiles; [
    common-utils
    search
    shell
    vcs
    editors
    data
    go
    node
    python
    rust
  ];
}
```

### Headless / Agentic

For CI, autonomous agent loops, or any context where SSH, editors, and prompts
are unnecessary.

```nix
# environments/headless.nix
{ pkgs, profiles, mkDevContainer, ... }:

mkDevContainer {
  name = "devenv-headless";
  enableSSH = false;
  profiles = with profiles; [
    common-utils
    search
    vcs
    data
    python
  ];
}
```

### Project-Specific

Extends the general environment with project-specific dependencies.

```nix
# environments/examples/python-project.nix
{ pkgs, profiles, mkDevContainer, ... }:

mkDevContainer {
  name = "devenv-myproject";
  enableSSH = true;
  profiles = with profiles; [
    common-utils
    search
    shell
    vcs
    editors
    python
    data
  ];
  extraPackages = with pkgs; [
    postgresql
    redis
  ];
}
```

### Special-Purpose

For tailored environments with unique requirements. Same mechanism as
project-specific, but may omit common profiles or include unusual packages.

```nix
# environments/examples/custom.nix
{ pkgs, profiles, mkDevContainer, ... }:

mkDevContainer {
  name = "devenv-custom";
  enableSSH = false;
  profiles = with profiles; [
    common-utils
    vcs
  ];
  extraPackages = with pkgs; [
    # Whatever the special purpose requires
    terraform
    packer
    ansible
  ];
}
```

---

## The Flake

```nix
# flake.nix
{
  description = "Nix-based composable development environments";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-24.11";
    nixpkgs-unstable.url = "github:NixOS/nixpkgs/nixpkgs-unstable";
  };

  outputs = { self, nixpkgs, nixpkgs-unstable }: let
    supportedSystems = [ "x86_64-linux" "aarch64-linux" ];

    unstableOverlay = system: final: prev: let
      unstable = nixpkgs-unstable.legacyPackages.${system};
    in {
      dolt = unstable.dolt;
      # opencode = unstable.opencode;  # when available in nixpkgs
      # Add fast-moving packages here
    };

    forAllSystems = f: nixpkgs.lib.genAttrs supportedSystems (system:
      f (import nixpkgs {
        inherit system;
        overlays = [ (unstableOverlay system) ];
      })
    );

    loadProfiles = pkgs: {
      common-utils = import ./profiles/common-utils.nix { inherit pkgs; };
      search       = import ./profiles/search.nix { inherit pkgs; };
      shell        = import ./profiles/shell.nix { inherit pkgs; };
      vcs          = import ./profiles/vcs.nix { inherit pkgs; };
      editors      = import ./profiles/editors.nix { inherit pkgs; };
      data         = import ./profiles/data.nix { inherit pkgs; };
      go           = import ./profiles/go.nix { inherit pkgs; };
      node         = import ./profiles/node.nix { inherit pkgs; };
      python       = import ./profiles/python.nix { inherit pkgs; };
      rust         = import ./profiles/rust.nix { inherit pkgs; };
      cloud        = import ./profiles/cloud.nix { inherit pkgs; };
      containers   = import ./profiles/containers.nix { inherit pkgs; };
    };

  in {
    packages = forAllSystems (pkgs: let
      profiles = loadProfiles pkgs;
      mkDevContainer = import ./lib/mkDevContainer.nix {
        inherit pkgs;
        lib = pkgs.lib;
      };
    in {
      general = import ./environments/general.nix {
        inherit pkgs profiles mkDevContainer;
      };
      headless = import ./environments/headless.nix {
        inherit pkgs profiles mkDevContainer;
      };
    });

    # Dev shell for working on devenv-nix itself
    devShells = forAllSystems (pkgs: {
      default = pkgs.mkShell {
        packages = with pkgs; [ nix-prefetch-docker dive skopeo ];
      };
    });
  };
}
```

---

## Runtime Model

The container runtime model is unchanged from the original devenv.

### Image Build and Load

```bash
# Build produces a script that streams the image
nix build .#general
# Load into Docker
$(./result) | docker load
```

### Container Lifecycle

```bash
# Start persistent container
docker run -d --name devenv-myproject \
  -v devenv-data:/home/devuser/.local/share \
  -v devenv-cache:/home/devuser/.cache \
  -v devenv-state:/home/devuser/.local/state \
  -v "$HOME/.config/bash:/home/devuser/.config/bash:ro" \
  -v "$HOME/.config/nvim:/home/devuser/.config/nvim:ro" \
  -v "$HOME/.config/starship.toml:/home/devuser/.config/starship.toml:ro" \
  -v "$HOME/.config/git:/home/devuser/.config/git:ro" \
  -v "$(pwd):/home/devuser/project:rw" \
  devenv:latest sleep infinity

# Attach sessions
docker exec -it devenv-myproject bash

# Headless execution
docker exec devenv-myproject <command>
```

### Persistent Volumes

| Volume | Mount Point | Purpose |
|---|---|---|
| `devenv-data` | `~/.local/share` | Plugins, parsers, tool databases |
| `devenv-cache` | `~/.cache` | Package caches |
| `devenv-state` | `~/.local/state` | Logs, history, session state |

### Dotfile Bind Mounts

Until dotfiles are converted to Nix / Home Manager, configuration is
bind-mounted read-only from the host:

- Bash config
- Neovim config
- Starship config
- Git config
- GitHub CLI config
- OpenCode config
- SSH agent socket (`$SSH_AUTH_SOCK`)

---

## Entrypoint

The entrypoint carries over from the original devenv. It runs under tini
(PID 1) and handles:

1. Start SSH server (if enabled)
2. Resolve repo/project root
3. Start dolt SQL server (if `.beads/dolt/` exists) — non-blocking, fail-safe
4. Health check (30s timeout)
5. `exec sleep infinity`

Future consideration: extract dolt server into a Docker Compose sidecar service.

---

## Comparison with Original Architecture

| Aspect | Original (Docker) | Nix |
|---|---|---|
| Package sources | apt, curl, GitHub releases, pipx, npm, rustup | nixpkgs (single source) |
| Tool isolation | One Dockerfile per tool, `COPY --from` | One `.nix` per profile, `buildEnv` merges |
| Reproducibility | Pinned Ubuntu, latest tool versions | `flake.lock` pins entire dep tree |
| Version bumps | Manual per-tool URL updates | `nix flake update` (atomic) |
| Build output | Docker layer tarball in layer cache | Stream to stdout, no store tarball |
| Dotfile management | Read-only bind mounts from host | Same (bind mounts), HM later |
| Inter-tool deps | Manual (ripgrep → jq image) | Automatic (Nix resolves all deps) |
| Image composition | `FROM` chains + `COPY --from` | Function: `mkDevContainer { profiles = [...]; }` |
| Adding a tool | Write Dockerfile + update aggregator | Add package name to a profile list |
| Channel strategy | N/A (apt + manual) | Stable default, unstable overlay for select packages |
| Architecture | Hardcoded in Dockerfiles | Defined once in `flake.nix`, derived everywhere |

---

## What Stays the Same

- Persistent container model (`sleep infinity` + `docker exec`)
- XDG volumes (`devenv-data`, `devenv-cache`, `devenv-state`)
- Security model (non-root user, SSH localhost-only, project dir is only rw mount)
- Container naming convention (`devenv-<parent>-<basename>`)
- Entrypoint pattern (tini → entrypoint.sh → optional services → sleep)
- Dotfile bind mounts (for now)

---

## Future Considerations

- **Home Manager integration** — When host dotfiles are converted to Nix, add
  `home-manager` as a flake input and build dotfiles into the image. The
  `mkDevContainer` function already anticipates this (add a `homeModules`
  parameter).
- **Separate CLI tool** — `jorgengundersen/devenv-cli` to handle container
  lifecycle, volume management, and project detection. Points to this repo for
  environment definitions.
- **Docker Compose for sidecars** — Dolt server, databases, and other services
  as sidecar containers instead of entrypoint processes.
- **Custom derivations** — For packages not in nixpkgs (beads, copilot-cli,
  opencode), define custom derivations in a `packages/` directory and add them
  to the overlay.
- **Profile granularity tuning** — Experiment with finer or coarser groupings.
  The architecture supports any granularity without structural changes.
- **Cross-architecture builds** — When moving to Mac, add `aarch64-darwin` to
  `supportedSystems` (for host tooling) and use `aarch64-linux` for container
  images (Docker on macOS runs Linux containers).
