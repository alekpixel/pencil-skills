# pencil-skills

**Three Claude Code skills for designing in [Pencil](https://pencil.dev) (pen.dev) through its MCP server.** Screens built from your design system, and design systems built from your code, without the usual failed calls and collapsed layouts.

[English](#english) · [Français](#français)

Unofficial community project. Not made or endorsed by pen.dev.

---

## English

### Why

Pencil ships an MCP server (`pencil`) that lets an agent read and write `.pen` files by running small JavaScript snippets. It works well once you know its rules: its layout model is not CSS, text is invisible without a fill, ids are never yours to set, a copied component becomes an instance, and a failed call must be patched rather than resent. These skills put those rules and a sound order of work in front of the agent.

### The skills

| Skill | What it does |
| --- | --- |
| `pencil-use` | The prerequisite. How the five MCP tools fit together, the `execute` API, layout and text sizing, variables and themes, components, instances and slots, a pre-flight checklist and a table of common errors. |
| `pencil-generate-design` | Builds or updates a page, screen, modal or panel from code or a description. Inventories the file's components and variables first, maps code tokens to Pencil variables, then builds section by section with a check after each one. Covers dark mode, breakpoints and empty or error states. |
| `pencil-generate-library` | Builds a design system in the right order: discovery, variables and light and dark themes, a foundations board, components one at a time with their variants, then QA and an optional `.lib.pen` library. Explains how variants, booleans, icon swaps and slots map onto Pencil's model. |

### Requirements

- [Claude Code](https://claude.com/claude-code)
- Pencil desktop app (or its IDE extension) running, with its MCP server enabled in Claude Code
- The target `.pen` file open in Pencil

### Install

As a Claude Code plugin:

```
/plugin marketplace add alekpixel/pencil-skills
/plugin install pencil-skills@pencil-skills
```

Or copy the three folders from `skills/` into `~/.claude/skills/`.

### Usage

Ask in plain words; the skills trigger on their own:

```
Design the settings page from src/app/settings/page.tsx in Pencil, light and dark.
```

```
Import the CSS variables from src/styles/globals.css into the open .pen file,
then build Button, Input and Badge components with their variants.
```

### Rules the skills keep

- Never read `.pen` files with file tools: they are encrypted, only the MCP reads them.
- No canvas write before the file has been inventoried and the plan shown.
- Colors, spacing and fonts come from variables; screens use component instances.
- One section or one component per call, checked before the next.
- A user checkpoint between phases of a design system build.
- Ask before generating AI images; prefer the project's real assets.

### License

MIT, see [LICENSE](LICENSE).

---

## Français

### Pourquoi

Pencil fournit un serveur MCP (`pencil`) qui permet à un agent de lire et d'écrire des fichiers `.pen` en lançant de petits scripts JavaScript. Ça marche bien quand on connaît ses règles : sa mise en page n'est pas du CSS, un texte sans remplissage est invisible, on ne choisit jamais les identifiants, un composant copié devient une instance, et un appel raté se corrige par un patch au lieu d'être renvoyé. Ces skills donnent ces règles à l'agent, avec le bon ordre de travail.

### Les skills

| Skill | Ce qu'il fait |
| --- | --- |
| `pencil-use` | Le prérequis. Comment les cinq outils du MCP s'articulent, l'API d'`execute`, la mise en page et la taille des textes, les variables et les thèmes, les composants, les instances et les slots, une liste de contrôle avant chaque appel et un tableau des erreurs courantes. |
| `pencil-generate-design` | Construit ou met à jour une page, un écran, une modale ou un panneau à partir du code ou d'une description. Fait d'abord l'inventaire des composants et des variables du fichier, relie les tokens du code aux variables de Pencil, puis construit section par section avec une vérification après chacune. Couvre le mode sombre, les largeurs d'écran et les états vides ou d'erreur. |
| `pencil-generate-library` | Construit un design system dans le bon ordre : repérage, variables et thèmes clair et sombre, planche des fondations, composants un par un avec leurs variantes, puis contrôle qualité et bibliothèque `.lib.pen` en option. Explique comment variantes, options, changement d'icône et slots se traduisent dans Pencil. |

### Prérequis

- [Claude Code](https://claude.com/claude-code)
- L'application Pencil (ou son extension d'IDE) ouverte, avec son serveur MCP activé dans Claude Code
- Le fichier `.pen` visé ouvert dans Pencil

### Installation

Comme plugin Claude Code :

```
/plugin marketplace add alekpixel/pencil-skills
/plugin install pencil-skills@pencil-skills
```

Ou copier les trois dossiers de `skills/` dans `~/.claude/skills/`.

### Utilisation

Demander en langage courant ; les skills se déclenchent seuls :

```
Dessine dans Pencil la page de réglages de src/app/settings/page.tsx, en clair et en sombre.
```

```
Importe les variables CSS de src/styles/globals.css dans le fichier .pen ouvert,
puis crée les composants Button, Input et Badge avec leurs variantes.
```

### Les règles qu'ils tiennent

- Ne jamais lire un `.pen` avec les outils de fichiers : ils sont chiffrés, seul le MCP les lit.
- Aucune écriture sur le canevas avant l'inventaire du fichier et la présentation du plan.
- Couleurs, espacements et polices viennent des variables ; les écrans utilisent des instances de composants.
- Une section ou un composant par appel, vérifié avant le suivant.
- Un point de validation avec l'utilisateur entre les phases d'un design system.
- Demander avant de générer des images par IA ; préférer les vrais fichiers du projet.

### Licence

MIT, voir [LICENSE](LICENSE).
