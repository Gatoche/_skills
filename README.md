# _skills

Skills [Claude Code](https://claude.com/claude-code) personnels partagés entre projets wipiSoft.

Repo GitHub : <https://github.com/Gatoche/_skills>

## Qu'est-ce qu'un skill ?

Un skill est un dossier contenant un `SKILL.md` qui décrit (via frontmatter YAML) une
procédure que Claude Code peut exécuter automatiquement quand l'utilisateur la déclenche.

Le `SKILL.md` contient :
- Un **frontmatter** (`name`, `description`) qui sert au matching automatique avec la requête
  de l'utilisateur
- Le **corps** (Markdown) avec le workflow détaillé, les pièges, les références à la source
  de vérité (HOWTO, code de référence, etc.)

## Skills disponibles

### [`wipisoft-module-converter/`](wipisoft-module-converter/)

Transforme une app C#/.NET 8 existante (WPF ou console) en **Module** ou **ModuleService**
wipiSoft selon les conventions du SDK partagé dans
[`_SDKs/Modules/`](../_SDKs/Modules/).

Source de vérité : les deux HOWTO du SDK
([Module](../_SDKs/Modules/Wps.Module.Sdk/HOWTO.md),
[ModuleService](../_SDKs/Modules/Wps.ModuleService.Sdk/HOWTO.md)). Le skill orchestre la
procédure : modifie le csproj, ajoute le bootstrap App.xaml.cs, crée le Description.md
embedded, build et reporte.

Déclencheurs typiques : « transforme cette app en Module wipiSoft », « convertis en ModuleService »,
« intègre cette app au host wipiSoft », « ajoute le SDK Module ».

## Comment installer un skill côté Claude Code

Les skills de ce dossier sont rendus disponibles à Claude Code via une référence dans la
config du projet (`.claude/skills.json` ou similaire selon la version), ou via un symlink
vers `~/.claude/skills/`.

Voir la doc Claude Code pour la procédure d'installation à jour.

## Licence

Projet propriétaire — wipiSoft
