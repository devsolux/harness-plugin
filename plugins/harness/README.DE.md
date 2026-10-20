# Harness-Plugin für Claude Code und Codex

![Harness](assets/logo.svg)

**Sprachen:** [English](README.md) · **Deutsch** · [Türkçe](README.TR.md)

Harness misst, ob eine Änderung an den Anweisungen eines Coding-Agents (`CLAUDE.md`,
`AGENTS.md` und Skills) Claude Code oder Codex die eigenen Prüfungen eines Projekts, an
denen der Agent zuvor gescheitert ist, zuverlässig bestehen lässt. In Ihrem Projekt
ändert sich nichts, bis Sie einen geprüften Kandidaten übernehmen, indem Sie seinen
Digest eintippen, und jede Übernahme lässt sich zurückrollen.

## Warum Harness

Das Modell ist gegeben: Claude oder GPT können Sie nicht neu trainieren. Was Sie steuern,
ist das System darum herum, also die Anweisungen und Skills, die Ihr Agent in jeder
Sitzung lädt. Die meisten Teams ändern dieses System nach Gefühl: eine Regel ergänzen,
ein- oder zweimal ausprobieren, behalten, wenn es geholfen zu haben scheint. Dieser Beleg
ist anekdotisch. Es ist eine einzelne Stichprobe ohne Vergleichsbasis, die CLI hat sich
womöglich zwischendurch aktualisiert, und persönliche Hooks, MCP-Server und Plugins
beeinflussen das Ergebnis. Wirkungslose Regeln bleiben stehen, und jede Zeile wird in
jeder Sitzung geladen.

Harness macht aus einer solchen Änderung ein kontrolliertes Experiment. Die unveränderte
Baseline und ein Kandidat bearbeiten dieselbe Aufgabe in derselben fixierten Umgebung,
wiederholt, und nach jedem Lauf folgen dieselben deterministischen Prüfungen. Die Antwort
ist entweder eine nachgewiesene Verbesserung mit einem exakten Diff zur Prüfung durch Sie
oder **No demonstrated improvement** (keine nachgewiesene Verbesserung).

### Mit und ohne Harness

| | Ohne Harness | Mit Harness |
| --- | --- | --- |
| Beleg | Ein oder zwei manuelle Versuche | Baseline gegen Kandidat, wiederholt, gleiche Prüfungen |
| Zufall des Modells | Unbemerkt („diesmal hat es geklappt“) | Wiederholungen müssen übereinstimmen, sonst keine Aussage |
| Umgebung | Ihr Alltags-Setup; CLI-Version, Hooks und MCP-Server variieren | Fixierte CLI, leere Konfiguration, kein Netzwerk |
| Sicherheit | Der Agent arbeitet in Ihrem echten Projekt | Eine private Kopie in der Sandbox; Ihr Projekt bleibt bis zur Übernahme unberührt |
| Modellnutzung | Unbegrenzt und nicht erfasst | Harte Limits für Requests, Sitzungen, Tokens und Zeit |
| Wenn nichts hilft | Die Anweisungsdatei wächst trotzdem | Nichts ändert sich, und das Ergebnis sagt es |
| Rückgängig machen | `git revert`, falls man daran denkt | `harness rollback`, gebunden an den angezeigten Diff |
| Was bleibt | Eine Regel mit unbekannter Wirkung | Eine Regel mit Beleg und der Test, der sie definiert |

### Wann sich Harness lohnt

- **Es lohnt sich** bei wiederkehrenden Fehlern des Agents, die ein Test erkennen kann,
  bei Anweisungsdateien, die ein Team teilt, bei Konventionen, die eingehalten werden
  müssen, und wenn viele Sitzungen pro Woche die Kosten eines Laufs amortisieren.
- **Es lohnt sich nicht**, wenn der Agent die Aufgabe bereits löst (dann kann sich nichts
  verbessern, das Ergebnis ist *No demonstrated improvement*), wenn keine
  deterministische Prüfung die Erwartung ausdrücken kann, wenn der Fehler an den
  Fähigkeiten des Modells statt an den Anweisungen liegt, oder bei einmaligen Aufgaben.

## Funktionsweise

```text
You: project + task + approved files + acceptance checks + limits
                           |
        Preflight: sandbox, trusted host CLI, limits, credential, checks
                           |
   Core: baseline -> Propose -> Probe -> Compose -> Select
                           |
   Native sessions (Claude Code or Codex CLI) in Bubblewrap, no network
                 |                                   |
     loopback forwarder  -->  provider gateway (holds the credential,
                              caps requests, meters tokens and cost)
                           |
   Independent validation: fresh baseline and candidate rollouts + approved checks
                           |
   Eligibility: improved / unchanged / regressed / indeterminate
                           |
   You review the exact diff -> explicit adoption -> safe rollback
```

1. **Preflight** verweigert den Start ohne funktionierende Sandbox, vertrauenswürdige
   Host-CLI, vollständige Limits, Zugangsdaten und Prüfungen, die in der Sandbox laufen
   können.
2. **Kern.** Der Propose–Probe–Compose-Kern führt die unveränderte Baseline aus, lässt
   den Host Änderungen an den Anweisungen vorschlagen, erprobt und kombiniert sie und
   wählt einen finalen Kandidaten. Bei `k=1` ist diese Auswahl trivial.
3. **Unabhängige Validierung** führt frische, abwechselnde Rollouts von Baseline und
   Kandidat aus; nach jedem Rollout laufen die freigegebenen Prüfungen in einer Sandbox
   ohne Netzwerk.
4. **Eignung.** Ein Kandidat ist nur geeignet, wenn alle freigegebenen Prüfungen für ihn
   bestehen, mindestens ein Fehlschlag der Baseline behoben ist, nichts regressiert und
   die Wiederholungen übereinstimmen. Andernfalls meldet Harness **No demonstrated
   improvement**. Eine Auswahl ist nie ein Beleg für eine Verbesserung.
5. **Übernahme** entscheiden allein Sie: Sie prüfen den exakten Diff und tippen seinen
   Digest in Ihr Terminal. Ein Rollback ist an einen Digest derselben Art gebunden.

Weder das Modell noch ein Plugin-Skill kann Ausgaben autorisieren, den Scope erweitern
oder Ihr Live-Projekt ändern. Unterstützte Anweisungsoberflächen, beschränkt auf die
Dateien, die Sie aufführen: `CLAUDE.md` und `.claude/skills/<name>/SKILL.md` für Claude
Code; `AGENTS.md` und `.agents/skills/<name>/SKILL.md` für Codex.

## Was das Plugin enthält

Drei Skills, ein Referenzdokument, diese README in drei Sprachen und zwei Icons für den
Eintrag im OpenAI-Plugin-Verzeichnis. Die Skills und das Referenzdokument sind auf
Englisch. Es gibt keine Hooks, MCP-Server, Befehle, Agents oder ausführbaren Dateien.

- `setup`: prüft die installierte Runtime, führt ihren Doctor und eine Offline-Demo aus
  und zeigt eine Projektkonfiguration zur Vorschau.
- `refine`: zeigt eine Verfeinerung als Trockenlauf ohne Modellaufrufe, führt eine erst
  aus, nachdem Sie die Ausgaben freigegeben haben, oder bewertet einen vorhandenen
  Kandidaten.
- `review`: zeigt die Nachweise eines Laufs und den exakten Diff des Kandidaten, erklärt,
  ob er geeignet ist, und begleitet eine freigegebene Übernahme, einen Rollback oder eine
  Wiederherstellung.
- `references/runtime-setup.md`: wie die Runtime installiert, geprüft, aktualisiert und
  entfernt wird.

## Was es ausführt, sendet und abruft

Die Skills lassen den Assistenten das separat installierte Kommandozeilenprogramm
`harness` (Python 3.12, Apache-2.0) auf Ihrem Rechner ausführen, zum Beispiel:

```bash
harness --version
harness doctor --json
harness demo --mode parallel --k 1 --json
harness init --project PROJECT_PATH --json
```

- Das Plugin selbst enthält keinen Code, sendet keine Daten und kontaktiert keinen
  Dienst.
- Die Installation der Runtime lädt `harness-1.0.0-py3-none-any.whl` und `SHA256SUMS`
  aus dem Release `v1.0.0` von https://github.com/devsolux/harness-plugin herunter. Der
  Setup-Skill prüft die Prüfsumme und fragt, bevor er etwas installiert.
- `harness refine --allow-model-calls` führt die gewählte Host-CLI (Claude Code oder
  Codex) in der Sandbox auf einer Kopie der freigegebenen Projektdateien aus. Diese CLI
  sendet die Aufgabe und das, was sie liest, über ein lokales Harness-Gateway an ihren
  Anbieter (Anthropic oder OpenAI); das Gateway hält die Zugangsdaten außerhalb der
  Sandbox und setzt Limits für Requests, Tokens und Kosten durch. Ohne diesen Schalter
  macht Harness keine Modellaufrufe.
- `harness login claude` und `harness login codex` starten die Anmeldung des jeweiligen
  Anbieters in Ihrem Terminal und speichern das erhaltene Token für das Gateway.
- Unter macOS oder unter Linux ohne Bubblewrap baut `harness-docker build` ein lokales
  Linux-Image. Dafür lädt es ein per Digest fixiertes Ubuntu-24.04-Basisimage,
  Ubuntu-Pakete und fixierte Builds von Claude Code und Codex herunter. Das Image wird
  nie hochgeladen.
- Läufe, Nachweise und Sicherungen bleiben in einem lokalen Statusverzeichnis. Harness
  hat keine Telemetrie; sein einziger Netzwerk-Client ist das Gateway, das die Requests
  der Host-CLI weiterleitet.

## Voraussetzungen

- Python 3.12 und ein isolierter Tool-Installer wie `uv`.
- Linux ohne Root-Rechte mit Bubblewrap für Bewertung und Verfeinerung in der Sandbox,
  oder Docker für das Linux-Workspace-Image unter macOS.
- Für die modellgestützte Verfeinerung: Claude Code oder Codex systemweit installiert und
  Ihre eigene Anmeldung bei Claude.ai oder ChatGPT oder ein API-Schlüssel, den Sie
  exportieren.

## Status

Version 1.0.0. Die Offline-Runtime, beide echten Host-CLIs gegen einen
Loopback-Anbieter und eine Live-Verfeinerung mit Claude Code samt Übernahme und Rollback
wurden geprüft. Die Abnahme auf einem unterstützten Linux-Host ohne Root-Rechte
außerhalb der CI steht noch aus; beaufsichtigen Sie echte Läufe daher.

## Lizenz

Apache-2.0. Der vollständige Text steht in `LICENSE`.
