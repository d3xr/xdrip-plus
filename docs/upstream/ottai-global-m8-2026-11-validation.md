# Ottai Global M8 — 2026-11-16 follow-up to upstream PR #736

## Hardware outcome and CI

In the user's iOS fork `d3xr/xdrip-plus`, **7.2.1 build 4256** was used on a **physical Ottai M8 Global** for (1) unlinking a previous account-bound sensor, (2) validating a new sensor, and (3) starting/activating the new sensor. The user reported all three actions succeeded. This is a firsthand user confirmation, **not independently captured device/API logs**. Continuous readings, accuracy, background reconnect, NFC wake and 30-day/post-original-expiry endurance were not verified by this report.

CI: [run #43813441659](https://github.com/d3xr/xdrip-plus/actions/runs/43813441659) — **265 selected XCTest cases, 0 failures**, Simulator app/extensions and unsigned iOS IPA built.

## Independent fixes beyond PR #736

1. **Global session profile:** `appName=ottai`, `packageName=com.ottai.seas`, consistent mobile `deviceId` and request signatures; new sign-in required when replacing earlier `ottai-seas` / watch sessions. Android source: [JugglucoNG 758d35f](https://github.com/ctqvva/JugglucoNG/commit/758d35f006c4f1f9cb48ae0b296b5249c4bab1ed).
2. **Global login request:** email `thirdLoginByPassword`, username `variousLogin` / `type=3`, and required `uuid`, `source=5` fields on both routes. Missing fields produced an HTTP-200 business refusal in the first Swift backport.
3. **Distinct new-sensor path:** V3 validation before activation, separate read-only restoration of existing sensors, no implicit activation or bind replay.
4. **Account already bound:** `AppUser_AlreadyBinding` leads to a read-only current-binding lookup, explicit MAC-checked unlink confirmation, exactly one `PUT /deviceBind/unBindDevice` and read-after-write reconciliation.
5. **Unbind reason:** Global rejected `unbindType=0` (`解绑类型输入不合法`); `unbindType=13` (`SyaiUnbindReason.endedEarly`) succeeded in the user's real scenario. Unlink can irreversibly end ownership and prevent re-pairing the retired sensor. Never unbind automatically.
6. **Recovery/safety:** sign-in/session guards, no swapping another account's cached keys, durable bind/activation checkpoints, no repeating uncertain physical sensor writes, specific stage diagnostics.

## Proposal for upstream review

Original donor PR: [JohanDegraeve/xdripswift #736](https://github.com/JohanDegraeve/xdripswift/pull/736) by [luborjurena](https://github.com/luborjurena).

Existing follow-up: [luborjurena/xdripswift #1](https://github.com/luborjurena/xdripswift/pull/1), branched from the original author's Ottai work. **On 2026-11-16, it was explicitly updated to [`c742d57`](https://github.com/d3xr/xdrip-plus/commit/c742d57c1da55f53d591a490a465ab4cea45770f)** by porting 10 Ottai implementation/test files and only the Ottai device-UI section from the verified full fork; the contributor PR still needs independent CI and review. No unrelated GS1, AiDex, styling, signing or full-fork workflows were included. Merging this PR into the original #736 is controlled by its author/upstream maintainers.

Please retain author and upstream credits (original Swift PR #736 and JugglucoNG). No patient-identifying MACs, credentials, raw BLE recordings or private signing keys are part of this documentation.
