<p align="center">
  <img src="https://raw.githubusercontent.com/heonny/egui-glass/main/assets/branding/readme-logo.png" alt="egui_glass" width="640">
</p>

[English](https://github.com/heonny/egui-glass/blob/main/README.md) | **Deutsch** | [한국어](https://github.com/heonny/egui-glass/blob/main/README.ko.md)

[![CI](https://github.com/heonny/egui-glass/actions/workflows/ci.yml/badge.svg)](https://github.com/heonny/egui-glass/actions/workflows/ci.yml)
[![crates.io](https://img.shields.io/crates/v/egui_glass.svg)](https://crates.io/crates/egui_glass)
[![API docs](https://docs.rs/egui_glass/badge.svg)](https://docs.rs/egui_glass)

Glasflächen für [egui](https://github.com/emilk/egui), angelehnt an Apples *Liquid Glass*:
Kantenbrechung, Milchglas-Unschärfe, Tönung und ein dünnes Facetten-Highlight, auf der GPU
über `egui-wgpu` gezeichnet. Enthalten sind Container, Kapselflächen-Buttons, Glas-Slider,
Werkzeugleisten, Hell/Dunkel-Vorlagen und optional ein Live-Hintergrund, der egui-Inhalt
bricht. Die Entwicklungsversion blendet Hover und Druck bei Buttons und Slidern kurz über;
das veröffentlichte 0.1.5 schaltet die Zustände sofort um.

![demo](https://raw.githubusercontent.com/heonny/egui-glass/main/docs/demo.png)

## Schnellstart

Rust 1.95 oder neuer, egui/eframe 0.37 und wgpu 30. Der Renderer muss **wgpu** sein; glow
wird nicht unterstützt. Die Features `wayland` und `x11` unten schalten die Fensterverwaltung
unter Linux ein. Ein wgpu-fähiger Grafikadapter ist nötig. Mit `cargo new glass-app` anfangen,
die Abhängigkeiten durch die folgenden ersetzen, das Rust-Beispiel nach `src/main.rs` legen
und `cargo run` ausführen. Es werden keine Assets und kein lokaler Checkout dieses
Repositories gebraucht.

```toml
[dependencies]
eframe = { version = "0.37", default-features = false, features = ["accesskit", "clipboard", "default_fonts", "wgpu", "wayland", "x11"] }
egui = "0.37"
egui_glass = "0.1.5"
```

```rust
use eframe::egui;
use egui_glass::{Glass, GlassToolbar};

struct App;

impl App {
    fn new(cc: &eframe::CreationContext<'_>) -> Self {
        let rs = cc.wgpu_render_state.as_ref().expect("eframe must use the wgpu renderer");
        egui_glass::init(rs, 1);
        // A gradient makes refraction visible; a uniform colour looks almost flat.
        let mut wallpaper = egui::ColorImage::filled([256, 256], egui::Color32::WHITE);
        for y in 0..256 {
            for x in 0..256 {
                wallpaper[(x, y)] = egui::Color32::from_rgb(70 + x as u8 / 2, 130, 180 + y as u8 / 4);
            }
        }
        egui_glass::set_backdrop(&cc.egui_ctx, rs, &wallpaper);   // what the glass refracts
        Self
    }
}

impl eframe::App for App {
    fn ui(&mut self, ui: &mut egui::Ui, _frame: &mut eframe::Frame) {
        egui::CentralPanel::default().frame(egui::Frame::NONE).show(ui, |ui| {
            egui_glass::show_backdrop(ui, ui.max_rect());
            egui::Area::new(egui::Id::new("card"))
                .default_pos(egui::pos2(40.0, 40.0)).movable(true).constrain(false)
                .show(ui.ctx(), |ui| {
                Glass::default().show(ui, |ui| {
                    ui.heading("Hello glass");
                    ui.label("Drag this card to see the background change.");
                    let _ = ui.button("Continue"); // ordinary controls inside glass
                });
            });
            egui::Area::new(egui::Id::new("bar"))
                .default_pos(egui::pos2(40.0, 200.0)).movable(true).constrain(false)
                .show(ui.ctx(), |ui| {
                GlassToolbar::default().show(ui, |ui| {
                    let _ = ui.button("Undo");
                    let _ = ui.button("Share");
                });
            });
        });
    }
}

fn main() -> eframe::Result {
    let options = eframe::NativeOptions { renderer: eframe::Renderer::Wgpu, ..Default::default() };
    eframe::run_native("glass", options, Box::new(|cc| Ok(Box::new(App::new(cc)))))
}
```

`init` läuft einmal pro Renderer; `set_backdrop` lädt einmal oder wenn sich das Bild ändert;
`show_backdrop` läuft in jedem Frame vor dem Glas. Die `1` an `init` entspricht der
Standard-MSAA (`NativeOptions::multisampling = 0` bedeutet eine Probe). Wenn MSAA eingeschaltet
ist, dieselbe Probenzahl an beide Stellen übergeben. Den Renderer so lange am Leben halten wie
die Oberfläche.

## Was das Glas sieht

| Hintergrund | Was durch das Glas scheint | Aufwand und Einsatz |
|---|---|---|
| Statisches Bild, wie oben | Nur das registrierte Bild an dieser Bildschirmposition | Einmal hochladen; Seitenleisten, Karten und Dialoge über einem Verlauf oder Hintergrundbild |
| `LiveBackdrop` | Seiteninhalt aus dem Closure, einschließlich Text | Zusätzliches Layout und GPU-Rendering in jedem Frame; schwebende Steuerelemente über wechselndem Inhalt |

Liegt zum Beispiel eine Logzeile über einem statischen Hintergrundbild, zeigt Glas über dieser
Zeile weiterhin **das Hintergrundbild**, nicht eine unscharfe Logzeile. Ein höherer `blur`
ändert daran nichts. Statisches Glas ist keine Fenstertransparenz des Betriebssystems und kann
weder den Desktop noch andere Apps abtasten.

Für einen Live-Hintergrund ein `LiveBackdrop` in der App halten. Es nach `init` anlegen, vor
`set_backdrop` oder der Registrierung nativer Texturen. Das ersetzt das Zeichnen des statischen
Hintergrunds; das Glas bleibt außerhalb des Seiten-Closures:

```rust
// In the creation closure, after init (rs is the wgpu render state):
let live = egui_glass::LiveBackdrop::new(rs, None); // default fonts

// In App::ui, using the stored live value:
live.run(ui, egui::Color32::WHITE, |page| {
    page.label("This text is part of the refracted page.");
});
egui::Area::new(egui::Id::new("overlay"))
    .fixed_pos(egui::pos2(20.0, 20.0)).constrain(false)
    .show(ui.ctx(), |ui| {
        egui_glass::Glass::panel().show(ui, |ui| { ui.label("Floating glass"); });
    });
```

Das Seiten-Closure läuft für das sichtbare und das Offscreen-Layout, und noch einmal, wenn egui
einen weiteren Durchgang anfordert: Dateischreiben, Netzwerkkaufrufe, Zustandswechsel und andere
Nebenwirkungen bleiben außerhalb. Ein Live-Hintergrund pro Renderer. Für eigene Schriften
`Some(font_definitions)` übergeben und sie mit dem Hauptkontext abgleichen. Bilder auf der Seite
müssen Texturen nutzen, die mit `egui_glass::register_native_texture` registriert wurden
(`set_backdrop` tut das für sein Bild); gewöhnliche egui-Textur-Uploads werden nicht automatisch
gespiegelt.

Die Entwicklungsversion kennt eine Qualität des Live-Hintergrunds pro Frame (noch nicht
veröffentlicht):

```rust
live.run_with_quality(ui, egui::Color32::WHITE, egui_glass::LiveBackdropQuality::Balanced, |page| {
    page.label("This text is part of the refracted page.");
});
```

`Full` nutzt die native Auflösung, `Balanced` 0,75× Breite und Höhe, `Performance` 0,5×.
`run` bleibt bei `Full`. Niedrigere Stufen verringern Offscreen-Pixel und Mipmaps, Seite und
Glasgeometrie bleiben in nativer Auflösung. Kleiner gebrochener Text wird weicher; der zusätzliche
Layout-Aufwand bleibt. Umfang der Messung und lokale Ergebnisse stehen in der
[Benchmark-Anleitung](https://github.com/heonny/egui-glass/blob/main/docs/performance.md).

## Glas in eine bestehende App

Initialisierung und Abhängigkeiten von oben verwenden. Mit Navigationsflächen oder kurzen
Overlays anfangen; dichten Text und Code auf deckenden Flächen lassen, wenn der Kontrast zählt.
Glas ersetzt weder die Ereignisbehandlung noch die Persistenz der Anwendung.

### Seitenleiste und deckender Inhalt

Dieser Code gehört in `App::ui`. Einen Hintergrund über die ganze Wurzel-Oberfläche zeichnen,
**bevor** Panels belegt werden, damit jede Fläche dieselbe Bildschirmabbildung nutzt.
Transparente äußere Panel-Rahmen lassen den Hintergrund durch. `Glass` bringt sein eigenes
Padding mit (standardmäßig 16 Punkte); einen zweiten gepolsterten Rahmen nur hinzufügen, wenn
beide Ränder gewollt sind.

```rust
egui_glass::show_backdrop(ui, ui.max_rect());
egui::Panel::left("navigation")
    .default_size(220.0)
    .frame(egui::Frame::NONE)
    .show(ui, |ui| {
        egui_glass::Glass::panel().inner_margin(egui::Margin::same(12)).show(ui, |ui| {
            ui.set_min_size(ui.available_size()); // fill the panel, even with little content
            ui.heading("Projects");
            egui::ScrollArea::vertical().show(ui, |ui| {
                for name in ["Frontend", "API", "Worker"] {
                    let _ = ui.button(name);
                }
            });
        });
    });
egui::CentralPanel::default()
    .frame(egui::Frame::new().fill(egui::Color32::WHITE))
    .show(ui, |ui| { ui.label("Opaque log or editor content"); });
```

### Modal mit Scrollen und Schließen

Das vorhandene Flag für das offene Modal und die Aktionsbehandlung behalten. Am Modal
`Frame::NONE` setzen und das Glas hineinlegen; das Modal blockiert die Eingabe weiterhin und
schließt bei Klick daneben oder Escape. Die innere `ScrollArea` scrollt das Formular, die
Fußzeile bleibt sichtbar.

```rust
// In App::ui, after drawing the page. `open` is your &mut bool modal state.
if *open {
    let response = egui::Modal::new(egui::Id::new("settings"))
        .frame(egui::Frame::NONE)
        .show(ui.ctx(), |ui| {
            egui_glass::Glass::panel().inner_margin(egui::Margin::same(20)).show(ui, |ui| {
                ui.set_width(320.0);
                ui.heading("Settings");
                let body_height = (ui.ctx().content_rect().height() - 160.0).max(80.0);
                egui::ScrollArea::vertical().max_height(body_height).show(ui, |ui| {
                    for n in 1..=20 { ui.label(format!("Setting {n}")); }
                });
                ui.button("Close").clicked()
            }).inner
        });
    if response.should_close() || response.inner { *open = false; }
}
```

### Vorhandenes Steuerelement-Theme behalten

`Glass::show` und `GlassToolbar::show` geben Kind-Steuerelementen flache Darstellung:
transparente Ruhefüllung, Kapselecken, geänderte Vordergrund- und Auswahlfarben, keine
Widget-Rahmen. Das gilt nur für ihre Kind-Oberfläche. In der Entwicklungsversion (noch nicht
veröffentlicht) behält `.preserve_theme(true)` die Buttons, Validierungsfarben und den
Fokus-Stil der App:

```rust
egui_glass::Glass::panel().preserve_theme(true).show(ui, |ui| {
    let _ = ui.button("Uses the app's theme");
});
```

`GlassToolbar` kennt dieselbe Option. Der Standard ist `false`, also flache Kind-Darstellung.
Die Option behält die Darstellung der Steuerelemente; Glasmaterial, Padding und Abstände der
Werkzeugleiste kommen weiter vom Glas-Container. Im veröffentlichten **0.1.5** den Stil
**vor** dem Betreten des Containers sichern:

```rust
let app_style = ui.style().clone();
egui_glass::Glass::panel().show(ui, |ui| {
    ui.set_style(app_style);
    let _ = ui.button("Uses the app's theme");
});
```

Eine Glastönung wählen, die zu den wiederhergestellten Textfarben passt; ein helles Theme auf
dunklem Glas stellt auch dessen dunklen Text wieder her. Innerhalb von Glas-Containern
gewöhnliche egui-Steuerelemente verwenden. `GlassButton` malt eine weitere Glasfläche und gehört
deshalb auf den Hintergrund, nicht verschachtelt in `Glass` oder `GlassToolbar`.

## Materialien und Steuerelemente

| API / Vorlage | Einsatz |
|---|---|
| `Glass::default()` / `GlassStyle::regular()` | Kleine Milchglas-Karten, zurückhaltende Facette und Halo |
| `Glass::panel()` / `GlassStyle::panel()` | Große Flächen und Seitenleisten mit ruhigen Kanten |
| `GlassStyle::clear()` | Stärkere Linse über bildreichem Inhalt, wenig Milchglas |
| `GlassStyle::dark()` / `panel_dark()` | Dunkle Gegenstücke; mit einem dunklen Hintergrund kombinieren |
| `GlassButton::new("Continue").show(ui)` | Eigenständige Kapsel-Aktion |
| `GlassToolbar::default().show(ui, closure)` | Werkzeugleiste mit gewöhnlichen egui-Steuerelementen darin |
| `GlassSlider::new(&mut value, 0.0..=100.0).show(ui)` | Eigenständiger Slider mit Glas-Griff und editierbarem Wert |
| `paint_glass(ui, rect, &style)` | Eigene Fläche; vor dem Vordergrund zeichnen |

Das Material ändern, ohne das Widget-Layout anzufassen:

```rust
let style = egui_glass::GlassStyle {
    blur: 20.0,
    refraction: 4.0,
    chromatic: 0.0,
    corner_radius: 12.0,
    tint: egui::Color32::from_rgba_unmultiplied(255, 255, 255, 180),
    ..egui_glass::GlassStyle::panel()
};
egui_glass::Glass::new(style).show(ui, |ui| { ui.label("A quiet sheet"); });
```

Höheres Tint-Alpha mischt mehr Tönungsfarbe bei; ein hohes weißes Tint verbessert den
Textkontrast, lässt den Hintergrund aber weniger durchscheinen. Unschärfe und Brechung sind in
logischen Punkten gemessen. Für dunkle Oberflächen eine dunkle Vorlage und ein passendes
Host-Theme bzw. einen passenden Hintergrund nehmen. Für weniger Transparenz in der App einen
gewöhnlichen deckenden `egui::Frame` wählen statt Glas zu zeichnen.

## Fehlersuche

In der Entwicklungsversion animieren `GlassButton` und `GlassSlider` Hover und Druck über die
lokale egui-`Style::animation_time`, gedeckelt bei 120 ms. `.animate(false)` am Widget oder die
Host-Animationszeit `0.0` schalten sofort um. Klicks, Fokus, Slider-Werte und Griffpositionen
aktualisieren sich weiter sofort.

| Symptom | Prüfen |
|---|---|
| Kein `wgpu_render_state` / kein Glas | eframe-Feature `wgpu` einschalten und `Renderer::Wgpu` wählen; glow wird nicht unterstützt |
| Panic bei `set_backdrop` oder Fehler bei der GPU-Probenzahl | Zuerst `init` aufrufen und die MSAA-Zahl des Renderers treffen |
| Glas wirkt flach oder weiß | Einen ungleichmäßigen Hintergrund registrieren, ihn vor dem Glas zeichnen und bei Bedarf das Tint-Alpha senken |
| Text hinter dem Glas erscheint nicht | Statisches Glas sieht nur sein Bild; für Seiteninhalt `LiveBackdrop` nutzen |
| Der Hintergrund verschwindet um Panels | Deckende Panel-Rahmen überdecken ihn; für diese äußeren Rahmen `Frame::NONE` |
| Buttons oder Auswahlfarben haben sich geändert | Glas setzt flache Kind-Darstellung; das Rezept zum Stil-Erhalt oben nutzen |
| Dialog-Abstand doppelt | Das Padding des äußeren Modal-Rahmens entfernen; Padding nur einmal an `Glass` geben |
| Eine schwebende Area springt im ersten Frame | `Area::constrain(false)` nutzen; die App legt die Position im Viewport selbst fest |
| Die Seite scrollt unter einem schwebenden Overlay nicht | egui schickt das Rad an die oberste Ebene; den scrollenden Inhalt auf diese Ebene legen oder das Rad in der App selbst weiterreichen |
| Bilder fehlen durch Live-Glas | Ihre nativen Texturen bei der Bibliothek registrieren, bevor die Live-Seite gezeichnet wird |
| Headless-Tests haben keinen GPU-Renderer | Inhalt mit einem deckenden Rahmen prüfen; Glas-Rendering getrennt mit wgpu prüfen |

## Vor dem Einsatz

Das ist eine **0.1-Bibliothek**: geeignet, um Glas in nativen egui-Apps auszuprobieren, mit
einer kleinen API, die sich vor 1.0 noch ändern kann. Rendering und Bedienung auf den
Zielgeräten selbst prüfen.

- Statisches Glas tastet das registrierte Bild ab; Widgets dahinter nimmt es nicht automatisch auf.
  `LiveBackdrop` zeichnet den Seiteninhalt noch einmal außerhalb des Bildschirms und kostet
  dadurch in jedem Frame Layout und GPU-Arbeit.
- Bibliothekstests laufen in CI unter Linux, macOS und Windows. Demo-Builds werden unter macOS
  und Windows geprüft. Das sind Build- und Testprüfungen, keine Zertifizierung des GPU-Renderings;
  Browser und Mobilgeräte deckt CI nicht ab.
- `GlassButton` unterstützt Tab-Navigation, Aktivierung mit Enter/Leertaste, einen sichtbaren
  Fokusrahmen und Button-Namen für Screenreader. Die AccessKit-Anbindung des Hosts einschalten
  (wie oben) und für Buttons nur aus einem Icon `.accessible_name("Back")` setzen. Eine deckende
  oder weniger transparente Alternative bleibt Sache der Anwendung; den Kontrast am tatsächlichen
  Hintergrund prüfen.
- Es gibt keine Standard-Features der Crate. `features = ["serde"]` einschalten, um `GlassStyle`
  zu serialisieren.

## Dokumentation

- [Anleitung](https://github.com/heonny/egui-glass/blob/main/docs/guide.md) — Glas in eine bestehende App: Hintergrund, Komponenten, Vorlagen, dunkles
  Theme, Glas über Live-Inhalt.
- [Referenz](https://github.com/heonny/egui-glass/blob/main/docs/reference.md) — jede Funktion und jedes `GlassStyle`-Feld, wie der Shader arbeitet,
  Grenzen, die Demo-App, die macOS-App bauen, Veröffentlichen.
- [docs.rs](https://docs.rs/egui_glass) — API-Dokumentation.
- [Mitwirken](https://github.com/heonny/egui-glass/blob/main/CONTRIBUTING.md) — Entwicklung, Fehlermeldungen und Release-Prüfungen.
- [Änderungsprotokoll](https://github.com/heonny/egui-glass/blob/main/CHANGELOG.md) — Änderungen für das nächste Release.
- [Prüfung](https://github.com/heonny/egui-glass/blob/main/docs/verification.md) — automatische GPU-Prüfungen und verbleibende native Tests.
- [Leistung](https://github.com/heonny/egui-glass/blob/main/docs/performance.md) — Qualitätsstufen, Benchmark-Befehle und Messgrenzen.

Anleitung, Referenz und die übrigen Dokumente bleiben auf Englisch. Deutsch gibt es nur in dieser README.

## Demo

```bash
cargo run -p egui_glass_demo          # sliders for every parameter, presets, export/import
cargo run -p egui_glass_demo --example minimal
make install                           # macOS: Egui Glass.app into /Applications
```

Die Entwicklungs-Demo hat **Copy Rust**, um das aktuelle Material als vollständigen
`egui_glass::GlassStyle`-Ausdruck zu kopieren. **Modified** markiert Abweichungen von der
gewählten Vorlage, **Reset** stellt diese Vorlage wieder her. Nach Import wird das importierte
Material zur Reset-Basis. Reset betrifft nur das Material; der Schalter für den Live-Hintergrund
bleibt unverändert.

## Lizenz

MIT. „Liquid Glass“ ist Apples Name für seine Gestaltungssprache; das hier ist eine unabhängige
Nachbildung des Aussehens und steht in keiner Verbindung zu Apple. Herkunft der Demo-Bilder und
des Brandings: [Asset-Herkunft](https://github.com/heonny/egui-glass/blob/main/docs/assets.md).
Diese Assets sind nicht in der Bibliotheks-Crate enthalten.

### Verwaltete Einbindung (Entwicklungsversion)

Die noch nicht veröffentlichte `GlassContext`-API fasst Initialisierung und Ressourcen zusammen:

```rust
let glass = egui_glass::GlassContext::new(ctx, rs, 1)?;
glass.set_fonts(font_definitions);
glass.set_backdrop(&wallpaper)?;
// Store `glass` in application state.
```

Sie ersetzt `init` und `LiveBackdrop::new`. Live-Rendering läuft über `glass.live_backdrop()`.
Doppelte Initialisierung und ungültige Bilder liefern `GlassError`; ein ungültiger Upload lässt
den bisherigen Hintergrund stehen. Native Texturen über `glass.register_native_texture(&view)`
liefern klonbare, besitzende Handles: sie während des Renderns halten und den letzten Handle die
Registrierung in beiden Renderern freigeben lassen. Lebensdauer und MSAA stehen in der
[Referenz der verwalteten API](https://github.com/heonny/egui-glass/blob/main/docs/reference.md#managed-setup-development-version).
Die veröffentlichten Einbindungsbeispiele für 0.1.5 oben bleiben gültig.
