# operatics-marketplace

Claude Code Plugin-Marketplace mit einem Demo-Plugin als Vorlage.

## Struktur

```
.claude-plugin/marketplace.json      # Marketplace-Metadaten, listet alle Plugins
plugins/
  demo-plugin/
    .claude-plugin/plugin.json       # Plugin-Metadaten
    skills/
      demo-skill/
        SKILL.md                     # Der eigentliche Skill
```

## Nutzung in Claude Code

Marketplace lokal hinzufügen (Pfad zu diesem Repo):

```
/plugin marketplace add /pfad/zu/operatics-marketplace
```

Oder nach dem Push, per GitHub-URL:

```
/plugin marketplace add thorstenster/operatics-marketplace
```

Plugin installieren:

```
/plugin install demo-plugin@operatics-marketplace
```

## Neuen Skill hinzufügen

1. Ordner unter `plugins/demo-plugin/skills/<neuer-skill>/` anlegen.
2. Darin eine `SKILL.md` mit YAML-Frontmatter (`name`, `description`) und der Anleitung für den Skill erstellen.
3. Marketplace neu laden bzw. Plugin neu installieren, damit Claude Code den Skill erkennt.

Für ein weiteres eigenständiges Plugin: neuen Ordner unter `plugins/` anlegen (analog zu `demo-plugin`) und in `.claude-plugin/marketplace.json` eintragen.
