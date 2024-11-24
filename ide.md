# IDE (Helix Editor)

Helix apparently is better than vim/neovim. It has sensible defaults for LSP, DAP, formatter, highlight etc eliminating the need for configuring plugins.

```sh
HELIX_VERSION=24.07
sudo curl -L https://github.com/helix-editor/helix/releases/download/${HELIX_VERSION}/helix-${HELIX_VERSION}-x86_64.AppImage -o /usr/local/bin/hx
sudo chmod +x /usr/local/bin/hx
```

My entire `~/.config/helix/config.toml`; how little compared to vim/neovim plugin configs!!

```toml
theme = "dracula"# "monokai"

[editor]
line-number = "relative"
mouse       = false
cursorline  = true

[editor.cursor-shape]
insert = "bar"
normal = "block"
select = "underline"

[editor.file-picker]
hidden     = false
git-ignore = true

[keys.normal]
esc = ["collapse_selection", "keep_primary_selection"]
```

## Languages
```sh
# golang
GO_VERSION=1.23.3
curl -OL https://go.dev/dl/go${GO_VERSION}.linux-amd64.tar.gz
sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go${GO_VERSION}.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
echo 'export PATH=$PATH:$HOME/go/bin' >> ~/.bashrc
rm go${GO_VERSION}.linux-amd64.tar.gz

# rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# bun - javascript runtime
curl -fsSL https://bun.sh/install | bash

# python
sudo dnf install -y python3 python3-pip
```

## IDE tools (LSP, DAP, etc)
```sh
# build-essential
sudo dnf group install -y development-tools

# c/cpp/clang
sudo dnf install -y clang clang-tools-extra lldb

# golang
go install golang.org/x/tools/gopls@latest
go install github.com/go-delve/delve/cmd/dlv@latest

# rust
cargo install --git https://github.com/rust-analyzer/rust-analyzer rust-analyzer --force

# python
pip3 install python-lsp-server

# toml
cargo install taplo-cli --locked

# yaml
bun install -g yaml-language-server

# typescript
bun install -g typescript-language-server typescript

# protobuf
go install github.com/bufbuild/buf-language-server/cmd/bufls@latest

# bash
bun install -g bash-language-server

# markdown
curl -L https://github.com/artempyanykh/marksman/releases/download/2024-11-20/marksman-linux-x64 -o ~/bin/marksman
```

##
Code assitant. Options aren't as plenty (and smooth) as VSCode. Gotta experiment with available ones for Helix.
