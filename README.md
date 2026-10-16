# voidbr-xbps

O **xbps** do Void Linux com duas extensões mantidas pela VoidBR:
**hooks de transação no estilo pacman** e **downloads paralelos com
interface de progresso no estilo pacman**.

Não é um fork: o código-base é o xbps oficial
([void-linux/xbps](https://github.com/void-linux/xbps)), fixado num commit
do upstream, e todas as mudanças ficam em dois patches aplicados por cima.
Repositórios, formato `.xbps`, assinaturas e pkgdb continuam 100%
compatíveis com o Void Linux.

O pacote `voidbr-xbps` fornece e substitui `xbps` e `libxbps`
(`provides`/`replaces`/`conflicts`), então ele toma o lugar do xbps original
no sistema.

---

## O que muda em relação ao xbps original

### `0001-xbps-hooks-support.patch`: hooks e transações

**Hooks de transação**, no mesmo formato do `alpm-hooks(5)` do pacman, em
`/etc/xbps.d/hooks.d/*.hook`:

```ini
[Trigger]
Operation = Install
Operation = Upgrade
Type      = Package
Target    = linux*

[Action]
Description = Gerando initramfs...
When        = PostTransaction
Exec        = /usr/bin/dracut --regenerate-all --force
NeedsTargets
```

| Chave | Valores |
|---|---|
| `Operation` | `Install`, `Upgrade`, `Remove` (pode repetir) |
| `Type` | `Package` ou `Path` |
| `Target` | padrão glob (pode repetir) |
| `When` | `PreTransaction` ou `PostTransaction` |
| `Exec` | comando a executar (sem shell; para lógica, aponte para um script) |
| `NeedsTargets` | passa os alvos que casaram no stdin do comando, um por linha |

Um hook `PreTransaction` que falha aborta a transação; um `PostTransaction`
que falha só é registrado. Hooks `PostTransaction` podem chamar
`xbps-install` e `xbps-remove` sem travar o pkgdb. Comandos dentro de hooks
devem usar `-y`, porque ali não há terminal para responder perguntas.

**Novas opções no `xbps-install`:**

| Opção | O que faz |
|---|---|
| `-O`, `--clean-cache` | remove do cache os pacotes desatualizados |
| `-OO` | também remove os pacotes que não estão instalados |
| `-OOO` | esvazia o cache inteiro |
| `-o`, `--remove-orphans` | remove os pacotes órfãos |
| `-ff` | além de sobrescrever arquivos, permite trocar um pacote que conflita com/substitui o pedido |

**Comportamento das atualizações:**

- **Auto-atualização:** quando o próprio gerenciador precisa ser atualizado
  durante um `-Syu`, ele se atualiza e continua o resto do upgrade sozinho,
  com uma única confirmação.
- **Pacotes renomeados:** no `-Syu`, um pacote que sumiu dos repositórios é
  trocado automaticamente pelo pacote que o declara em `replaces=`.
- **Pacote virtual protegido:** um pacote que fornece outro (como o
  `voidbr-xbps` fornecendo o `xbps`) nunca é trocado pelo original sem
  querer; para forçar, use `-ff`.

### `0002-xbps-parallel-downloads.patch`: downloads paralelos e interface

**Downloads simultâneos**, de 1 a 16 (o padrão do xbps é 1). A prioridade,
da maior para a menor:

1. opção de linha de comando: `xbps-install --parallel-downloads 8 ...`
2. variável de ambiente: `XBPS_PARALLEL_DOWNLOADS=8 xbps-install ...`
3. configuração: `paralleldownloads=8` em `/etc/xbps.d/*.conf`

O `xbps-install` informa no início o valor em uso e de onde ele veio:

```
[*] Parallel downloads: 5 (xbps.d)
```

As assinaturas são verificadas depois de todos os downloads, como no pacman.
Se um download falha, os outros não começam e o erro é o mesmo do modo
sequencial.

**Interface no terminal**, no estilo pacman (exemplo ilustrativo):

```
 firefox-158.0_1             86MB   13MB/s 00:07 [■■■■■■■■■■■■■■■■■■■■] 100%
 breeze-6.8.0_1              37MB  7650KB/s 00:05 [■■■■■■■■■■■■■■⠹□□□□□]  71%
 Total (101/102)            579MB    56MB/s 00:10 [■■■■■■■■■■■■■■■■■■■⠹]  97%
(102/102) Verifying package integrity              [■■■■■■■■■■■■■■■■■■■■] 100%
(102/102) Collecting package files                 [■■■■■■■■■■■■■■■■■■■■] 100%
(  1/102) Unpacking firefox-158.0_1                [■■■■■■■■■■■■■■■■■■■■] 100%
(  2/102) Updating breeze-6.7.5_1 -> 6.8.0_1       [■■■■■■■■■■■■■■■■■■■■] 100%
(102/102) Configuring packages                     [■■■■■■■■■■■■■■■■■■■■] 100%
```

- Uma barra por download, mais o `Total`; todas as barras na mesma coluna.
- Uma linha por etapa (`Verifying package integrity`,
  `Collecting package files`) e uma por pacote ao desempacotar.
- `Configuring` numa linha só, que também vale para o `xbps-reconfigure`.
  Quando um pacote tem script INSTALL, a saída do script aparece abaixo da
  linha, sem embaralhar.
- O que está sendo processado tem um spinner animado (`⠋ ⠙ ⠹ ⠸ ...`);
  no console texto (`TERM=linux`) ou sem UTF-8, ele vira `| / - \` e a barra
  vira `[#####-----]`.
- Cores: verde, amarelo e cinza; desligadas com `NO_COLOR` ou `TERM=dumb`.

**Fora do terminal** (log, pipe, `dialog --progressbox`) ou com `-d`/`-v`, a
saída é a mesma do xbps original, uma linha por pacote, boa para log.

**Correções na libfetch** necessárias para o paralelo: estado de erro por
thread, callbacks serializados e inicialização segura entre threads,
validados com ThreadSanitizer.

---

## vinstall: wrapper para os comandos xbps-*

O [vinstall](https://github.com/voidlinuxbr/voidbr-vinstall) (pacote
`voidbr-vinstall`, já incluído na base da VoidBR) é um wrapper em Go que
reúne `xbps-install`, `xbps-query` e `xbps-remove` num único comando, com
atalhos no estilo pacman. É um pacote separado, mas foi feito para usar com
este xbps.

- **Repassa as flags:** tudo que o `xbps-install` aceita funciona igual no
  vinstall, inclusive o que este pacote acrescenta (`-O`, `-o`, `-ff`, as
  barras e os downloads paralelos).
- **Sugere pacotes:** se o nome não existe, busca nos repositórios
  (`xbps-query -Rs`) e oferece um menu para escolher.
- **sudo só quando precisa:** roda como usuário comum e só pede `sudo` na
  hora de alterar o sistema.

```sh
vinstall telegram          # instala (ou sugere, se o nome não existir)
vinstall -Syu              # atualiza o sistema
vinstall -X pacote         # remove
vinstall -XR pacote        # remove com as dependências órfãs
vinstall -Ss termo         # busca nos repositórios
vinstall -F ifconfig       # qual pacote instalado tem esse arquivo
vinstall -FR ifconfig      # idem, nos repositórios
vinstall -Li               # lista os instalados
vinstall -Lo               # lista os órfãos
vinstall -Scc              # limpa o cache e remove órfãos
vinstall --history         # histórico de transações
```

A lista completa está em `vinstall -h`.

> **Downloads paralelos pelo vinstall:** o `xbps-install` chamado pelo
> vinstall já usa o `paralleldownloads=` do `/etc/xbps.d/xbps.conf`. Para
> trocar o valor numa execução, funcionam as três formas (a partir do
> vinstall 1.6.7):
>
> ```sh
> vinstall --parallel-downloads 8 -Syu
> vinstall --parallel-downloads=8 -Syu
> XBPS_PARALLEL_DOWNLOADS=8 vinstall -Syu
> ```
>
> Como o `sudo` limpa o ambiente, o vinstall converte a variável em
> `--parallel-downloads=8`; por isso o `xbps-install` mostra
> `(--parallel-downloads)` como origem.

---

## Configuração padrão do pacote

| Arquivo | Conteúdo |
|---|---|
| `/etc/xbps.d/xbps.conf` | configuração ativa da VoidBR, com `paralleldownloads=5`; preservada nas atualizações (`conf_files`) |
| `/usr/share/xbps.d/xbps.conf` | exemplo comentado com todas as opções, incluindo `paralleldownloads` |

Para voltar ao comportamento sequencial: `paralleldownloads=1`.

---

## Como o pacote é construído

O `pkgfile/PKGFILE` é processado pelo
[pkgmake](https://github.com/voidlinuxbr/voidbr-pkgmake):

1. clona o xbps do upstream no commit fixado em `_xbps_commit`;
2. aplica `pkgfile/0001-*.patch` e `pkgfile/0002-*.patch`, nessa ordem
   (o 0002 depende do 0001); qualquer falha aborta o build;
3. compila com `./configure --enable-rpath --prefix=/usr --sysconfdir=/etc`;
4. instala junto os arquivos de configuração e as chaves dos repositórios
   da VoidBR.

```sh
cd pkgfile
pkgmake
```

Os patches também ficam salvos em
[voidbr-make-xbps-hooks](https://github.com/voidlinuxbr/voidbr-make-xbps-hooks),
com o script `rmake` para compilar e testar fora do pacote.

---

## Licença

xbps: BSD-2-Clause (veja `usr/share/licenses/xbps/`).
Patches e arquivos da VoidBR: MIT (veja `LICENSE`).
