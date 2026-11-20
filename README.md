# Noctalia: buscar sessões do Claude Code abertas no kitty

Busca uma sessão do [Claude Code](https://docs.claude.com/en/docs/claude-code) pelo nome no launcher do [Noctalia](https://docs.noctalia.dev/noctalia/launcher/) e foca a janela dela no [Niri](https://github.com/YaLTeR/niri).

Digite `/cc` no launcher para ver só as sessões do Claude. Continue digitando para filtrar (`/cc relat`) e aperte Enter: a janela recebe o foco, mesmo em outro workspace.

Cada resultado também mostra o **perfil** (diretório de config) que a sessão usa:

```text
Relatório mensal [original]       #12 · original · workspace dev
Refatorar autenticação [work]     #34 · work · workspace dev
```

## Sintoma

Com várias sessões do Claude Code abertas, cada uma numa janela do kitty, é difícil achar a sessão certa no meio das outras janelas e workspaces.

## Ambiente verificado

- Arch Linux
- Niri (sessão Wayland)
- Noctalia v5.4.1
- kitty, com uma janela do sistema por sessão do Claude
- Claude Code
- `jq` e `pstree` (pacote `psmisc`)

## Diagnóstico: o que descobrimos

### 1. O Claude Code coloca o nome da sessão no título do terminal

Cada sessão aparece com um ícone de status seguido do nome:

```text
✳ Relatório mensal          ← sessão parada
◑ Refatorar autenticação    ← sessão trabalhando (o ícone muda)
```

Como cada sessão é uma janela própria do kitty, o Niri enxerga esse título:

```bash
niri msg -j windows | jq -r '.[] | select(.app_id=="kitty") | "\(.id)\t\(.title)"'
```

Por isso não foi preciso ligar o controle remoto do kitty (`allow_remote_control`).

### 2. O título sozinho não basta para identificar o Claude

Outras janelas do kitty (nvim, shell) também têm títulos. Para filtrar só o Claude, o script usa o PID que o Niri informa para cada janela e procura um processo `claude` dentro dela:

```bash
pstree -p <pid> | grep -q 'claude('
```

### 3. Dá para descobrir qual perfil (`.claude`) cada sessão usa

É comum ter mais de uma conta ou configuração do Claude Code, separadas por `CLAUDE_CONFIG_DIR` com aliases no `~/.zshrc`:

```bash
alias claude-work="CLAUDE_CONFIG_DIR=~/.claude-work claude"
alias claude-pessoal="CLAUDE_CONFIG_DIR=~/.claude-pessoal claude"
# "claude" puro usa ~/.claude
```

O título da janela não mostra isso, mas o ambiente do processo mostra. Com o PID do `claude` encontrado no passo anterior:

```bash
tr '\0' '\n' < /proc/<pid-do-claude>/environ | grep '^CLAUDE_CONFIG_DIR='
```

O script converte o valor num rótulo curto:

| `CLAUDE_CONFIG_DIR` | Rótulo |
| --- | --- |
| não definida | `original` (`~/.claude`) |
| `~/.claude-work` | `work` |
| `~/.claude-pessoal` | `pessoal` |

O rótulo vai no título do resultado (`Nome [work]`), então `/cc work` filtra as sessões daquele perfil, se o launcher incluir esse trecho na busca.

### 4. O launcher do Noctalia aceita fontes de resultado via script, sem plugin

A [documentação do launcher](https://docs.noctalia.dev/noctalia/launcher/) descreve entradas estilo dmenu em `[shell.launcher.dmenu.entry.<id>]`:

| Chave | Função |
| --- | --- |
| `command` | Roda via `/bin/sh -lc`. Cada linha da saída vira um resultado. Um TAB separa título e descrição. |
| `exec` | Roda ao selecionar. `{selection}` é a linha escolhida e `{query}` é o texto digitado. |
| `prefix` | Palavra de ativação, sem a barra (`cc` → `/cc`). |
| `label` / `glyph` | Nome e ícone ([Tabler](https://tabler.io/icons)) na lista de providers. |
| `global` | Se os resultados também aparecem na busca sem prefixo. |

Para lógica mais complexa existe a alternativa de plugin em Luau, com `[[launcher_provider]]` e as funções `onQuery`/`onActivate`. Para este caso, a entrada dmenu é suficiente.

### 5. Alternativa sem configurar nada

O provider nativo `/win` já busca pelo título e pelo app id das janelas. `/win Relatório` funciona, mas mistura as sessões do Claude com todas as outras janelas.

## Correção aplicada

### 1. Instalar dependências

```bash
sudo pacman -S --needed jq psmisc
```

### 2. Instalar o script

```bash
install -Dm755 bin/noctalia-claude-sessions ~/.local/bin/noctalia-claude-sessions
```

O script tem dois modos:

- `list`: imprime `Nome da sessão [perfil]<TAB>#id · perfil · workspace <nome>` para cada janela do kitty que roda o Claude. Ele tira o ícone de status e remove `"`, `'`, `` ` ``, `$` e `\` do nome, para não quebrar o comando `exec`.
- `focus "<linha>"`: tira o `#id` da linha e roda `niri msg action focus-window --id <id>`. Se a linha vier sem o `#id`, ele procura a janela pelo nome.

Teste:

```bash
~/.local/bin/noctalia-claude-sessions list
```

### 3. Fazer backup do config do Noctalia

```bash
cp ~/.config/noctalia/config.toml \
  ~/.config/noctalia/config.toml.bak-$(date +%Y%m%d%H%M%S)
```

### 4. Registrar o provider no launcher

Adicione ao **final** de `~/.config/noctalia/config.toml` (veja [`examples/config.toml`](examples/config.toml)):

```toml
[shell.launcher.dmenu.entry.claude]
command = "~/.local/bin/noctalia-claude-sessions list"
exec = "~/.local/bin/noctalia-claude-sessions focus '{selection}'"
label = "Sessões Claude"
prefix = "cc"
glyph = "robot"
global = false
```

Se o `/cc` não aparecer, reinicie o Noctalia para ele recarregar o config.

## Status da verificação

- ✅ `list` mostra só as janelas com Claude e deixa de fora outras janelas do kitty (ex.: nvim).
- ✅ A busca pelo nome encontra o id correto da janela.
- ✅ O perfil detectado (`original` e dois perfis via `CLAUDE_CONFIG_DIR`) bate com o alias usado para abrir cada sessão.
- ⚠️ O foco ao selecionar dentro do launcher ainda não foi testado de ponta a ponta.

## Limitações

- **Abas do kitty:** se várias sessões estiverem em abas da mesma janela, o Niri só enxerga o título da aba ativa. Nesse caso é preciso ligar `allow_remote_control` no kitty e usar `kitty @ ls` e `kitty @ focus-tab`.
- **Só Niri:** o foco usa `niri msg`. Em outros compositores, troque os comandos de listar e focar janelas.
- **Perfil:** o script lê o ambiente do momento em que o `claude` foi iniciado. Ele só funciona para processos do seu próprio usuário (o `/proc/<pid>/environ` de outros usuários não é legível).
- **Caracteres especiais:** aspas, `$`, crase e `\` são removidos do nome exibido. A documentação do Noctalia não diz se `{selection}` é escapado.

## Restauração

```bash
rm ~/.local/bin/noctalia-claude-sessions
```

Depois, remova o bloco `[shell.launcher.dmenu.entry.claude]` de `~/.config/noctalia/config.toml` ou restaure o backup:

```bash
cp ~/.config/noctalia/config.toml.bak-<data-hora> ~/.config/noctalia/config.toml
```

## Licença

MIT
