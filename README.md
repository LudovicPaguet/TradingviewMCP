# Claude vs TradingView

> Un pont local entre **Claude** et **TradingView Desktop**. Aucune API tierce, aucune
> donnée qui sort de ta machine : on parle directement à l'app via le **Chrome DevTools
> Protocol** (port `9222`). Claude lit ton graphique, le pilote, écrit du Pine Script,
> rejoue l'historique et te prépare un brief avant chaque session.

```
┌──────────┐   stdio    ┌─────────────┐   CDP :9222   ┌───────────────────────┐
│  Claude  │ ─────────► │ Serveur MCP │ ────────────► │ TradingView Desktop    │
│ (ou CLI) │ ◄───────── │  (Node.js)  │ ◄──────────── │ (Electron, ton compte) │
└──────────┘   JSON     └─────────────┘   évaluation  └───────────────────────┘
```

Tout tourne en local. Le serveur ne fait qu'injecter du JavaScript dans la fenêtre
TradingView déjà ouverte et relire ce qu'elle affiche — y compris les dessins Pine
(`line.new`, `label.new`, `box.new`, `table.new`) invisibles aux outils classiques.

---

## Sommaire

1. [Idée en une phrase](#idée-en-une-phrase)
2. [Prérequis](#prérequis)
3. [Installation](#installation)
4. [Brancher Claude (MCP)](#brancher-claude-mcp)
5. [Piloter depuis le terminal (CLI)](#piloter-depuis-le-terminal-cli)
6. [Le brief matinal & `rules.json`](#le-brief-matinal--rulesjson)
7. [Catalogue des 81 outils](#catalogue-des-81-outils)
8. [Dépannage](#dépannage)
9. [Structure du dépôt](#structure-du-dépôt)
10. [Avertissement & licence](#avertissement--licence)

---

## Idée en une phrase

TradingView n'expose pas d'API pour son app desktop. Mais l'app est une fenêtre
Electron — donc une instance Chrome. En la lançant avec `--remote-debugging-port=9222`,
on obtient un canal CDP par lequel un script Node peut **lire l'état réel du graphique
et agir dessus**. Ce dépôt empaquette 81 actions de ce type, exposées deux fois :

- **comme serveur MCP**, pour que Claude les appelle en langage naturel ;
- **comme CLI**, pour les scripter dans un terminal (sortie JSON, pipe-friendly).

---

## Prérequis

| | |
|---|---|
| **TradingView Desktop** | L'application native (pas le site web). Abonnement payant requis pour les données temps réel. |
| **Node.js** | Version 18 ou supérieure. |
| **Claude Code** | Pour les outils MCP. Facultatif si tu n'utilises que la CLI. |
| **OS** | macOS, Windows ou Linux. |

---

## Installation

```bash
# 1. Récupérer le code
git clone https://github.com/ludovicPaguet/claudeverstradingview.git ~/claudeverstradingview
cd ~/claudeverstradingview

# 2. Installer les dépendances (3 paquets, zéro build)
npm install

# 3. Exposer la commande globale (facultatif mais pratique)
npm link
```

Lancer ensuite TradingView **avec le port de debug ouvert** — un script par OS :

```bash
./scripts/launch_tv_debug_mac.sh      # macOS
scripts\launch_tv_debug.bat           # Windows
./scripts/launch_tv_debug_linux.sh    # Linux
```

> Si TradingView est déjà lancé normalement, ferme-le d'abord : le port de debug ne peut
> être ouvert qu'au démarrage du processus. Alternative sans script :
> l'outil `tv_launch` (MCP) ou `claudeverstradingview launch` (CLI) tentent la détection
> et le lancement automatiques.

Vérifie la connexion :

```bash
claudeverstradingview status        # attendu : { "cdp_connected": true, ... }
```

---

## Brancher Claude (MCP)

**Méthode recommandée (Claude Code)** — une seule commande :

```bash
claude mcp add claudeverstradingview -- node ~/claudeverstradingview/src/server.js
```

**Méthode manuelle** (Claude Desktop ou tout autre client MCP) — ajoute ce bloc à ta
configuration MCP, en remplaçant le chemin par le tien :

```json
{
  "mcpServers": {
    "claudeverstradingview": {
      "command": "node",
      "args": ["/chemin/absolu/vers/claudeverstradingview/src/server.js"]
    }
  }
}
```

Redémarre le client, puis demande à Claude :

> « Utilise `tv_health_check` pour vérifier que TradingView est connecté. »

Une fois branché, tu parles en langage naturel — Claude choisit les outils :

> « Analyse mon graphique : prix, indicateurs, niveaux Pine, et une capture. »
> « Passe sur ES1! en 15 minutes et ajoute le RSI. »
> « Rejoue le 1er mars 2025 et avance bougie par bougie. »

---

## Piloter depuis le terminal (CLI)

La CLI couvre les mêmes outils, sans Claude. Chaque commande **émet du JSON sur
`stdout`** — idéal avec `jq`. Codes de sortie : `0` succès, `1` erreur, `2` connexion CDP.

```bash
claudeverstradingview brief                 # brief matinal complet
claudeverstradingview status                # état de la connexion CDP
claudeverstradingview quote                 # dernier prix / OHLC / volume
claudeverstradingview state                 # symbole, TF, indicateurs + IDs
claudeverstradingview symbol BTCUSD         # changer de ticker
claudeverstradingview timeframe 15          # changer de résolution
claudeverstradingview ohlcv --summary       # stats compactes des bougies
claudeverstradingview values                # valeurs des indicateurs visibles
claudeverstradingview screenshot -r chart   # capture (full | chart | strategy_tester)
claudeverstradingview pane layout 2x2       # grille 4 graphiques
claudeverstradingview session get           # dernier brief sauvegardé

claudeverstradingview --help                # liste complète des commandes
claudeverstradingview <commande> --help     # options d'une commande
```

Exemple de composition avec `jq` :

```bash
claudeverstradingview ohlcv --summary | jq '.range, .change_pct'
```

> Sans `npm link`, remplace `claudeverstradingview` par `node src/cli/index.js` ou
> `npm run tv --`.

---

## Le brief matinal & `rules.json`

Le cœur du projet : une commande qui scanne ta watchlist, lit tous tes indicateurs et
renvoie des **données structurées** que Claude transforme en biais de session.

```bash
# Définir tes règles une fois pour toutes
cp rules.example.json rules.json
$EDITOR rules.json      # watchlist, critères de biais, règles de risque
```

`rules.json` est lu automatiquement par `morning_brief`. Puis, chaque matin :

```bash
claudeverstradingview brief
```

Claude applique tes règles et produit un tableau du type :

```
XAUUSD  | BIAIS: Baissier  | NIVEAU CLÉ: 3 280  | SURVEILLER: résistance Order Block 4H
XAGUSD  | BIAIS: Neutre    | NIVEAU CLÉ: 32,50  | SURVEILLER: structure en range
BTCUSD  | BIAIS: Haussier  | NIVEAU CLÉ: 83 000 | SURVEILLER: tenue au-dessus de l'OB daily

Global : prudence sur les métaux. BTC le plus solide des trois.
```

Garde une trace pour comparer les sessions :

- **Sauvegarder** — `session_save` (ou demande « sauvegarde ce brief »). Écrit dans
  `~/.ludovic/sessions/AAAA-MM-JJ.json`.
- **Relire** — `session_get` renvoie le brief du jour, sinon celui de la veille.

---

## Catalogue des 81 outils

Tous les outils renvoient `{ "success": true|false, ... }`. Les IDs d'entités viennent de
`chart_get_state` et sont valables pour la session courante seulement.

<details>
<summary><b>Brief matinal &amp; sessions</b> (3)</summary>

`morning_brief` · `session_save` · `session_get`
</details>

<details>
<summary><b>Lire le graphique</b> (chart + data + quote, 16)</summary>

**État & prix** — `chart_get_state` · `quote_get` · `data_get_ohlcv` (`summary: true` pour
le compact) · `data_get_study_values` · `data_get_equity`
**Graphiques Pine (invisibles aux outils standard)** — `data_get_pine_lines` ·
`data_get_pine_labels` · `data_get_pine_tables` · `data_get_pine_boxes`
**Stratégies & exécutions** — `data_get_strategy_results` · `data_get_trades` ·
`data_get_indicator`
**Symboles & marché** — `symbol_search` · `symbol_info` · `depth_get`
**Plage visible** — `chart_get_visible_range`
</details>

<details>
<summary><b>Contrôler le graphique</b> (chart + indicator + pane, 12)</summary>

`chart_set_symbol` · `chart_set_timeframe` · `chart_set_type` · `chart_manage_indicator`
(noms complets : « Relative Strength Index », pas « RSI ») · `chart_scroll_to_date` ·
`chart_set_visible_range` · `indicator_set_inputs` · `indicator_toggle_visibility` ·
`pane_set_layout` (`s`, `2h`, `2v`, `2x2`, `4`, `6`, `8`) · `pane_set_symbol` ·
`pane_focus` · `pane_list`
</details>

<details>
<summary><b>Watchlist</b> (2)</summary>

`watchlist_get` · `watchlist_add`
</details>

<details>
<summary><b>Pine Script</b> (12)</summary>

`pine_set_source` → `pine_smart_compile` → `pine_get_errors` → `pine_get_console`.
Aussi : `pine_compile` · `pine_check` · `pine_analyze` · `pine_get_source` (⚠ peut être
volumineux) · `pine_new` · `pine_open` · `pine_save` · `pine_list_scripts`
</details>

<details>
<summary><b>Replay (backtest manuel)</b> (6)</summary>

`replay_start` → `replay_step` / `replay_autoplay` → `replay_trade` (buy / sell / close) →
`replay_status` → `replay_stop`
</details>

<details>
<summary><b>Dessins & alertes</b> (8)</summary>

`draw_shape` (ligne, trendline, rectangle, texte) · `draw_list` · `draw_get_properties` ·
`draw_remove_one` · `draw_clear` · `alert_create` · `alert_list` · `alert_delete`
</details>

<details>
<summary><b>UI, onglets, layouts & captures</b> (22)</summary>

**Captures & lots** — `capture_screenshot` · `batch_run` (plusieurs symboles)
**Layouts** — `layout_list` · `layout_switch`
**Onglets** — `tab_new` · `tab_switch` · `tab_close` · `tab_list`
**Pilotage UI brut** — `ui_open_panel` · `ui_click` · `ui_type_text` · `ui_keyboard` ·
`ui_hover` · `ui_scroll` · `ui_mouse_click` · `ui_find_element` · `ui_evaluate` ·
`ui_fullscreen`
**Diagnostic** — `tv_launch` · `tv_health_check` · `tv_discover` · `tv_ui_state`
</details>

> Décompte vérifié sur `src/tools/` : **81** définitions `server.tool(...)`. Pour réduire
> le bruit en contexte : préfère `summary: true`, cible un indicateur précis avec
> `study_filter`, et évite `pine_get_source` sur les scripts complexes (200 Ko+).

---

## Dépannage

| Symptôme | Cause probable / correctif |
|---|---|
| `cdp_connected: false` | TradingView n'a pas été lancé avec `--remote-debugging-port=9222`. Relance-le via le script du dossier `scripts/`. |
| `ECONNREFUSED` | App fermée, ou port `9222` occupé/bloqué par un pare-feu. |
| Serveur MCP absent côté Claude | Vérifie le chemin dans la config MCP, puis redémarre le client. Avec Claude Code : `claude mcp list`. |
| `command not found` | Lance `npm link` depuis le dossier du projet (ou utilise `node src/cli/index.js`). |
| `No rules.json found` | `cp rules.example.json rules.json`, puis remplis-le. |
| Données qui semblent figées | TradingView charge encore le symbole/la TF — laisse une à deux secondes. |

---

## Structure du dépôt

```
src/
  server.js          point d'entrée MCP (stdio)
  connection.js      client CDP vers le port 9222
  core/              logique métier (chart, data, pine, replay, morning, …)
  tools/             enrobage MCP de chaque fonction core (les 81 server.tool)
  cli/               CLI : index.js + router.js + commands/*.js
scripts/             lanceurs TradingView (mac / win / linux) + utilitaires Pine
skills/              modes d'emploi par scénario (chart-analysis, pine-develop, …)
tests/               tests Node natifs (e2e, pine_analyze, cli)
rules.example.json   gabarit de règles à copier en rules.json
```

Lancer les tests :

```bash
npm test            # e2e + analyse Pine
npm run test:cli    # tests CLI seuls
```

---

## Avertissement & licence

Projet **non officiel**, sans affiliation avec TradingView Inc. ni Anthropic, PBC.
Fourni à des fins personnelles, éducatives et de recherche — utilisation à tes propres
risques, et dans le respect des [conditions d'utilisation de TradingView](https://www.tradingview.com/policies/).

Sous licence MIT — voir [LICENSE](LICENSE).
