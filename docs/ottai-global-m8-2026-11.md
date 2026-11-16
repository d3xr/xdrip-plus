# Ottai Global M8 — подтверждённый сценарий в xDrip+ 7.2.1 (4256)

**Дата:** 16 ноября 2026. **Основная ветка:** [`custom-build`](https://github.com/d3xr/xdrip-plus/tree/custom-build). **Исправления:** [`fix/ottai-activation-500`](https://github.com/d3xr/xdrip-plus/tree/fix/ottai-activation-500), итоговый код [`a51111d`](https://github.com/d3xr/xdrip-plus/commit/a51111d3af02bd08fc4dbb466996025d8320c180).

## Что удалось проверить

Пользователь **на физическом Ottai M8 Global** подтвердил успешное выполнение всех трёх шагов в iOS xDrip+ 4256:

1. Отвязка ранее привязанного сенсора от аккаунта Ottai Global (server-side unbind).
2. Проверка нового сенсора через `validateDeviceByMacV3`.
3. Активация нового физического сенсора из приложения.

Это сообщение о реальном успешном использовании, **не независимая запись BLE/API-трассы**. Не установлены длительность работы после исходного срока, точность глюкозы, устойчивость фонового соединения, последующая работа с Trio и поддержка 30 дней конкретной прошивкой. Тесты программного кода не заменяют этих наблюдений.

CI для 4256: [GitHub Actions #43813441659](https://github.com/d3xr/xdrip-plus/actions/runs/43813441659): **265 XCTest пройдены, 0 ошибок**, Simulator app, расширения и unsigned device IPA собраны. IPA из CI нельзя установить на iPhone без локальной Apple-подписи.

## Пошаговая настройка нового Ottai Global M8

1. В карточке Ottai/Syai выбери **Region: Global**. Войди в аккаунт через **Account**: после перехода на новый Global API требуется новый вход, старые токены профилей `ottai-seas` / `ottai-watch` повторно использовать нельзя.
2. Введи с коробки **12-символьный Sensor ID (MAC)** нового M8. **Choose sensor from account** — только выбор уже привязанного или исторического сенсора; новый в списке может отсутствовать.
3. Для **нового** сенсора выбери **Check new Global sensor**. Это предварительная проверка через V3, без команды физической активации. **Restore already activated sensor** / Fetch предназначен для восстановления уже привязанного сенсора.
4. Если появляется `AppUser_AlreadyBinding`, перейди в **Cloud account → Unlink previous cloud sensor** (либо **Review linked sensor** в предупреждении), проверь **MAC старого сенсора**. Отвязывай его **только если завершил его использование**, после явного подтверждения. Приложение повторно считывает текущую привязку, отправляет **один** запрос на отвязку и повторным чтением проверяет результат.
5. После подтверждённой отвязки повтори **Check new Global sensor**. Затем подключи именно новый физический M8 по Bluetooth и отдельно подтверди **Start sensor / Activate**. Активация запускает его срок, поэтому повторное нажатие для работающего сенсора запрещено.
6. После запуска проверь статус и показания. Не считай успешный `ACK` подтверждением продления срока. Для срока используются запись и read-back; фактическая работа за пределами обычного срока отдельно не подтверждена.

**Важно:** отвязка — изменение облачного владения сенсором и может окончательно пометить старый экземпляр как завершённый (после этого сервер может отказать в повторной привязке). Она **не отправляет BLE-команду остановки**, не очищает сохранённые ключи xDrip и не запускает новый M8. Если старый сенсор ещё подаёт показания, не отвязывай его ради теста. У приложения нет автоматического перебора вариантов отвязки или повторного `PUT` после неопределённого результата.

## Ошибки, воспроизведённые при разработке, и исправления

| Симптом | Причина / исправление | Где |
| --- | --- | --- |
| `500 NET_SECURITY_SYSTEM_ERROR`, в том числе замаскированная ошибка обновления приложения | Ottai Global перестал принимать запросы к device/bind через старый watch-профиль. Для текущего mobile API используются `appName=ottai`, `packageName=com.ottai.seas`, собственный согласованный deviceId, новый токен после входа. | [Android JugglucoNG 758d35f](https://github.com/ctqvva/JugglucoNG/commit/758d35f006c4f1f9cb48ae0b296b5249c4bab1ed) |
| `Sign in: HTTP 200 unknown` | В новой версии Swift `thirdLoginByPassword` отсутствовали поля `uuid` и `source=5`. Восстановлен полный контракт; для username используется `variousLogin` с `type=3`. | Коммиты 4253/4254 |
| `New sensor validation (V3): AppUser_AlreadyBinding` | В облаке уже привязан другой сенсор. Это не обязательно означает, что физический сенсор ещё работает. Добавлены отдельная проверка текущего MAC и добровольная отвязка с подтверждением. | Коммит 4255 |
| `解绑类型输入不合法` (недопустимый тип отвязки), `HTTP 200 / Error` | Старый Swift отправлял `unbindType=0`. Для **добровольного досрочного завершения** выбран `unbindType=13` по [SyaiUnbindReason.endedEarly](https://github.com/nightscout/SyaiKit/blob/816108980650863b637934367dc5e35212001956/SyaiKit/Server/Account/SyaiUnbindReason.swift). Сервер Ottai Global принял его в пользовательском сценарии 4256. | Коммит 4256 |

### Контракт Global

- Host: `https://seas.ottai.com`, защищённый API `/cgm/security/app/server` через native encrypted transport.
- Профиль: `ottai` / `com.ottai.seas`; заголовки и подписи связаны с текущим deviceId и сессией.
- Email: `/user/login/thirdLoginByPassword`. Username: `/user/login/variousLogin`, `type=3`. Оба запроса содержат `uuid`, `source=5`, зашифрованное `encryptInfo`, `apiToken`, `signature`.
- Новый M8: `POST /device/validateDeviceByMacV3` только проверяет серверные метаданные. Binding/ключи и физическая BLE-активация имеют отдельные условия; не повторять неопределённые `bindV3` и `ACTIVATE` автоматически.
- Отвязка старого: `GET /deviceBind/getBindDevice` → сверка MAC и сессии → **один** `PUT /deviceBind/unBindDevice` с `{mac, deviceType:"cgm", unbindType:13}` → подтверждающий `GET`. При неясном ответе нельзя повторять PUT без сверки состояния.

## Ветки, источники и границы переносимости

- **Текущая полная версия:** `d3xr/xdrip-plus:custom-build`, 7.2.1 (4256).
- **Исходная Swift-интеграция:** [JohanDegraeve/xdripswift #736](https://github.com/JohanDegraeve/xdripswift/pull/736) (автор Lubor Jurena); в ней ещё не объединены все исправления из данного форка.
- **Follow-up к автору:** [luborjurena/xdripswift #1](https://github.com/luborjurena/xdripswift/pull/1) **обновлён 16.11.2026** до коммита [`c742d57`](https://github.com/d3xr/xdrip-plus/commit/c742d57c1da55f53d591a490a465ab4cea45770f). Перенесены только 10 файлов Ottai/тестов и секция Ottai в SwiftUI — без GS1, ребрендинга и личных Apple signing overrides. Сборку именно этой upstream-ветки необходимо проверить отдельно; решение о принятии остаётся за автором и сопровождающими.
- **Android:** [ctqvva/JugglucoNG](https://github.com/ctqvva/JugglucoNG), актуальные правки Global от 15 ноября 2026.
- Сборка на Mac: [профили подписи](build-profiles.md). Меняя локальный worktree, сохраняй прежние Team ID, Bundle ID, App Groups, чтобы не потерять контейнер приложения и Trio sharing. Не добавляй токены/ключи/идентификаторы пациента в Git.

Основные файлы реализации: `OttaiGlobalContract.swift` (login), `OttaiCloudClient.swift` (V3 / bind / unlink), `OttaiRegistry.swift` (сессия/состояние), `CGMOttaiTransmitter.swift` (BLE), `BluetoothPeripheralDetailState.swift` (UI), `OttaiGlobalLoginTests.swift`, `OttaiServerActivationTests.swift`.

