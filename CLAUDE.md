# zeversolar-esphome-gateway — CLAUDE.md

Public how-to: wiring and an ESPHome YAML for reading a Zeversolar TL3000 inverter's internal RS485 port with an ESP32-S3. **Superseded for Kieren's own inverter** by `esphome-zeversolar`
(that project found the protocol is AA55, **not Modbus**, so this repo's `modbus_controller` sensors are wrong; and the RS485 link on the inverter has since failed). Kept public as the
wiring/pinout reference.

## Forge (changed 2 Nov 2026)

**Gitea is primary; GitHub is a read-only push mirror and must never be pushed to directly** (the mirror force-syncs and prunes refs; Gitea syncs on every push and every 10 minutes).
Work on `https://git.kiztigs.com/kmbrimble/zeversolar-esphome-gateway` (`origin`, gt credential helper); use `gt`, not `gh`, for issues and PRs. `main` is protected: PR + the green
`CI / check (pull_request)` (`.gitea/workflows/ci.yml`: `esphome config` against a pinned ESPHome, with a throwaway `secrets.yaml`); butler-bot merges its own green PRs. GitHub has Issues, Wiki,
Projects, Discussions and PRs switched off. Nothing deploys from this repo.
