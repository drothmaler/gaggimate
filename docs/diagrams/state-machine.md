# Machine State Diagram (Modes)

This diagram models the machine modes tracked by `Controller::mode`
(see `src/display/core/constants.h`). The modes are:

| Value | Name           | Macro          |
|-------|----------------|----------------|
| 0     | Standby        | `MODE_STANDBY` |
| 1     | Brew           | `MODE_BREW`    |
| 2     | Steam          | `MODE_STEAM`   |
| 3     | Water          | `MODE_WATER`   |
| 4     | Grind          | `MODE_GRIND`   |

Mode transitions happen through `Controller::setMode(int)`
(`src/display/core/Controller.cpp:669`) or through the two helpers
`Controller::activateStandby()` / `Controller::deactivateStandby()`
(`src/display/core/Controller.cpp:647` / `:652`).

## Mermaid (editable / exportable)

The diagram below is written in [Mermaid] state-diagram syntax. GitHub renders
it inline; it can also be pasted into the [Mermaid Live Editor][live] to export
PNG / SVG / PDF.

[Mermaid]: https://mermaid.js.org/syntax/stateDiagram.html
[live]: https://mermaid.live

```mermaid
stateDiagram-v2
    direction LR

    [*] --> Startup : Controller boots
    Startup --> Standby : settings.startupMode == STANDBY\n(Controller.cpp:289)
    Startup --> Brew    : default startup mode

    state "MODE_STANDBY (0)" as Standby
    state "MODE_BREW (1)"    as Brew
    state "MODE_STEAM (2)"   as Steam
    state "MODE_WATER (3)"   as Water
    state "MODE_GRIND (4)"   as Grind

    %% --- Wake from Standby ---
    Standby --> Brew  : brew button / onWakeup / onBrewScreen\n(deactivateStandby, Controller.cpp:748, 752)
    Standby --> Brew  : AutoWakeup schedule match\n(AutoWakeupPlugin.cpp:64)
    Standby --> Brew  : HomeKit accessory ON\n(HomekitPlugin.cpp:96)
    Standby --> Steam : steam button pressed\n(handleSteamButton, Controller.cpp:788)
    Standby --> Water : onWaterScreen (UI)
    Standby --> Grind : onGrindScreen (UI)
    Standby --> Brew  : WebUI req:change-mode=BREW\n(WebUIPlugin.cpp:329)
    Standby --> Steam : WebUI req:change-mode=STEAM
    Standby --> Water : WebUI req:change-mode=WATER
    Standby --> Grind : WebUI req:change-mode=GRIND

    %% --- Between operating modes (UI screen switches) ---
    Brew  --> Steam : onSteamScreen / steam button\n(Controller.cpp:791, ui_events.cpp:59)
    Brew  --> Water : onWaterScreen (ui_events.cpp:53)
    Brew  --> Grind : onGrindScreen (ui_events.cpp:91)

    Steam --> Brew  : brew button (Controller.cpp:765)\nsteam button release, non-momentary (Controller.cpp:798)\nonBrewScreen / onMenuClick / onWakeup
    Steam --> Water : onWaterScreen
    Steam --> Grind : onGrindScreen

    Water --> Brew  : onBrewScreen / onMenuClick / onWakeup
    Water --> Steam : onSteamScreen
    Water --> Grind : onGrindScreen

    Grind --> Brew  : onBrewScreen / onMenuClick / onWakeup
    Grind --> Steam : onSteamScreen
    Grind --> Water : onWaterScreen

    %% --- Back to Standby (any non-standby mode) ---
    Brew  --> Standby : idle > standbyTimeout (Controller.cpp:350)\nBLE disconnect (Controller.cpp:142)\ncontroller error (Controller.cpp:162)\nOTA update (onOTAUpdate)\nautotune start (Controller.cpp:379)\nHomeKit accessory OFF (HomekitPlugin.cpp:98)\nonStandby (ui_events.cpp:75)
    Steam --> Standby : same triggers as Brew → Standby
    Water --> Standby : same triggers as Brew → Standby
    Grind --> Standby : same triggers as Brew → Standby

    %% --- WebUI can force any mode ---
    Brew  --> Grind : WebUI req:change-mode
    Brew  --> Water : WebUI req:change-mode
    Brew  --> Steam : WebUI req:change-mode

    note right of Standby
      Entering Standby always calls deactivate()
      and clears the heater setpoint.
      activateStandby() = setMode(STANDBY) + deactivate()
    end note

    note left of Brew
      deactivateStandby() = deactivate() + setMode(BREW).
      BREW is the default "operating" mode that the UI
      falls back to (onMenuClick, steam release, ...).
    end note
```

## PlantUML (alternate, also editable)

```plantuml
@startuml
hide empty description
skinparam state {
  BackgroundColor<<op>> #E8F4FD
  BackgroundColor<<idle>> #EEEEEE
}

[*] --> Standby : startupMode == STANDBY
[*] --> Brew    : default startup

state Standby <<idle>> : MODE_STANDBY (0)
state Brew    <<op>>   : MODE_BREW    (1)
state Steam   <<op>>   : MODE_STEAM   (2)
state Water   <<op>>   : MODE_WATER   (3)
state Grind   <<op>>   : MODE_GRIND   (4)

' Wake from Standby
Standby --> Brew  : brew button / onWakeup\nAutoWakeup / HomeKit ON
Standby --> Steam : steam button
Standby --> Water : onWaterScreen
Standby --> Grind : onGrindScreen

' Between operating modes
Brew  --> Steam : onSteamScreen / steam btn
Brew  --> Water : onWaterScreen
Brew  --> Grind : onGrindScreen

Steam --> Brew  : brew btn / steam release\nonBrewScreen / onMenuClick
Steam --> Water : onWaterScreen
Steam --> Grind : onGrindScreen

Water --> Brew  : onBrewScreen / onMenuClick
Water --> Steam : onSteamScreen
Water --> Grind : onGrindScreen

Grind --> Brew  : onBrewScreen / onMenuClick
Grind --> Steam : onSteamScreen
Grind --> Water : onWaterScreen

' Back to Standby
Brew  --> Standby : standby timeout / OTA / error\nBLE disconnect / autotune / HomeKit OFF
Steam --> Standby : (same triggers)
Water --> Standby : (same triggers)
Grind --> Standby : (same triggers)

' WebUI can force any mode
Brew  --> Standby : WebUI req:change-mode
Standby --> Brew  : WebUI req:change-mode
@enduml
```

## Source references

All mode transitions found via `setMode` / `activateStandby` / `deactivateStandby`:

| Trigger                                    | Source location                              | Target mode  |
|--------------------------------------------|----------------------------------------------|--------------|
| Startup (startupMode == STANDBY)           | `Controller.cpp:289`                         | STANDBY      |
| BLE disconnect                             | `Controller.cpp:142`                         | STANDBY      |
| Remote / controller error                  | `Controller.cpp:162`                         | STANDBY      |
| Standby idle timeout                       | `Controller.cpp:350`                         | STANDBY      |
| Autotune start                             | `Controller.cpp:378-379`                     | STANDBY      |
| OTA update                                 | `Controller.cpp:686-687`                     | STANDBY      |
| `activateStandby()` helper                 | `Controller.cpp:647`                         | STANDBY      |
| `deactivateStandby()` helper               | `Controller.cpp:652-654`                     | BREW         |
| Brew button in STANDBY                     | `Controller.cpp:747-748`                     | BREW         |
| Brew button in BREW (wake)                 | `Controller.cpp:750-752`                     | BREW         |
| Brew button in STEAM                       | `Controller.cpp:763-765`                     | BREW         |
| Steam button in STANDBY                    | `Controller.cpp:787-788`                     | STEAM        |
| Steam button in BREW                       | `Controller.cpp:790-791`                     | STEAM        |
| Steam button release (non-momentary)       | `Controller.cpp:796-798`                     | BREW         |
| UI: onBrewScreen                           | `ui_events.cpp:48`                           | BREW         |
| UI: onWaterScreen                          | `ui_events.cpp:53`                           | WATER        |
| UI: onSteamScreen                          | `ui_events.cpp:59`                           | STEAM        |
| UI: onWakeup                               | `ui_events.cpp:70`                           | BREW         |
| UI: onStandby                              | `ui_events.cpp:75`                           | STANDBY      |
| UI: onMenuClick                            | `ui_events.cpp:85`                           | BREW         |
| UI: onGrindScreen                          | `ui_events.cpp:91`                           | GRIND        |
| AutoWakeup schedule match                  | `AutoWakeupPlugin.cpp:64`                    | BREW         |
| HomeKit accessory ON                       | `HomekitPlugin.cpp:96`                       | BREW         |
| HomeKit accessory OFF                      | `HomekitPlugin.cpp:98`                       | STANDBY      |
| WebUI `req:change-mode`                    | `WebUIPlugin.cpp:329`                        | any          |
