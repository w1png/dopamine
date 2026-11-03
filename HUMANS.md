# HUMANS.md

## Current
- **Repo:** dopamine
- **Branch:** `fix/android-tv-ui`
- **Task:** Android TV — фокус и ввод frkn:// с пульта, адаптация интерфейса, ARM APK
- **Status:** Готово. Собраны APK для arm64-v8a и armeabi-v7a в `deploy/build/`. На Android TV 14 (arm64) проверены фокус, экранная клавиатура, отправка тестовой frkn:// ссылки на валидацию и добавление офлайн-профиля с пульта. На KIVI 4K Android TV (Android 11) предыдущая версия удалена, APK armeabi-v7a 4.8.14.70 установлен и запущен.
- **Updated:** 2026-11-04

## Done
- 2026-11-04 — На KIVI 4K Android TV через ADB заменён FRKN на тестовую сборку 4.8.14.70 (armeabi-v7a). Проверены SHA-256 APK на ТВ и холодный запуск.
- 2026-11-04 — Android TV: ТВ-компоновка стартового экрана, импорта и главного экрана, фокус и ввод frkn:// с пульта. Собраны и подписаны тестовым ключом оба ARM APK.
- 2026-10-23 — **настоящая причина split-tunnel**: `service/server/main.cpp` после `tunneldaemon` проваливался в `LocalServer localServer;` → tunnel-процесс (он же `dopamine-service.exe tunneldaemon "<config>"`) поднимал **второй WindowsDaemon**: второй firewall-движок, второй слушатель `\\.\pipe\frkn` и второе открытие драйвера. По логу: `Started as console application` → `Tunnel daemon service is starting` → через 1.2 с `Daemon created` + `Server path: \\.\pipe\frkn`, повтор каждые 2 с. Второй `create()` видел уже инициализированный драйвер, делал `resetDriver()` и стирал правила, которые только что залил сервис → сплит-туннель «включался» и тут же терялся, а дальше два процесса дрались за устройство, которое открывается эксклюзивно. Фикс: `return daemon.run(tokens);` — tunnel-процесс больше не строит LocalServer
- 2026-10-23 — `windowssplittunnel.cpp`: `CreateFileW` на драйвер теперь с `FILE_SHARE_READ|FILE_SHARE_WRITE` в обоих местах (было `dwShareMode=0`), в лог добавлен `GetLastError()`. Нужно как страховка на окно перезапуска сервиса, когда прошлый процесс ещё держит хендл
- 2026-10-23 — собран и установлен MSI: `build_win.sh` (sync рабочего дерева → сборка под schtasks → MSI на мак) → `msiexec /qn` через SYSTEM-задачу; в установленном `dopamine-service.exe` есть маркер нового кода, драйвер открывается
- 2026-10-23 — грабли скриптов хоста: (1) `build_win.sh` не удалял прошлый MSI, поэтому «готово» приходило через 8 с и забирался старый инсталлятор — теперь удаляет до старта задачи и падает, если не смог; (2) `deploy/run_windows_admin_task.ps1` падал на первом запуске (`schtasks /delete` по несуществующей задаче = terminating error при `ErrorActionPreference=Stop`); (3) он же писал лог/маркер в `$env:TEMP`, а `Set-Content -Encoding ASCII` превращал кириллический `C:\Users\куся\...` в `????` → лог не писался, вызывающий висел до таймаута. Лог/маркер теперь рядом со скриптом (`C:\setup`)
- 2026-10-23 — диагностика на железе: зависший демон держит WFP, и тогда **любой** новый демон виснет в `FwpmSubLayerGetByKey0` (`windowsfirewall.cpp:121`), т.е. после `kill -f` демона нужен reboot бокса, иначе сервис не поднимается
- 2026-10-22 — вин-билд-хост настроен и собирает: VS 2022 Build Tools, Qt 6.10.1 msvc2022_64, IFW 4.7, WiX 4.0.5 (+ UI/Util extensions), .NET SDK 8, CMake/Ninja/PortableGit в `C:\tools`, репо + 6 сабмодулей, zlib-пребилды. Скрипты: `deploy/setup_windows_build.ps1` (без elevation: портативные тулзы, Python/.NET в профиль, WiX через `dotnet tool`, PATH в user-scope), `deploy/setup_windows_admin.ps1` (VS через `vs_BuildTools.exe`, OpenSSH, power plan, Defender), `deploy/run_windows_admin_task.ps1` (админская часть как SYSTEM через schtasks — ssh на винде не elevated).
- 2026-10-22 — сборка из **рабочего дерева без коммита**: `~/Desktop/dopamine-win-bootstrap/sync_src.sh` собирает `git diff --binary` + untracked-исходники, накатывает на клон винды (`git checkout -- .` → `git apply`), затем сверяет `git diff | git hash-object` с обеих сторон — при совпадении сборка идёт ровно из этих исходников. `build_win.sh` = sync → сборка под schtasks → MSI на мак.
- 2026-10-22 — `windowssplittunnel.cpp`: `IP_ADAPTER_ADDRESSES_LH` **не содержит** `Ipv4IfIndex` (есть только `IfIndex`/`Ipv6IfIndex`), из-за чего файл вообще не компилировался (C2039 в двух местах). IPv4-индекс теперь берётся из `GetIpAddrTable()` по IPv4-адресу адаптера (`ipv4IfIndexFor`), `byIndex` ищет сначала в IPv6-пространстве индексов, затем в IPv4.

## Next
- [ ] Android TV: проверить на физическом ТВ импорт действующей frkn:// ссылки и подключение.
- [ ] ГЛАВНОЕ: прогон на железе после логина юзера на боксе (после reboot интерактивной сессии нет, AutoAdminLogon выключен). Подключиться, затем в логе ждём `Pushing new Ruleset` + `Split tunnel addresses` + `New Network Config Applied || new State: STATE_RUNNING`, а `curl.exe` (он в исключениях) должен отдавать ISP-адрес `79.139.146.253`, не exit-IP сервера `93.89.220.48`
- [ ] если после коннекта демон опять замолчит в логе (зависание внутри WFP) — смотреть, не делает ли второй демон `resetDriver()` поверх включённого драйвера; лечится только reboot бокса
- [ ] вторая половина (проверить на прогоне): если в логе появится `Split tunnel manager is not initialized` — `config.m_vpnDisabledApps` пустой. Сборка конфига: `vpnconnection.cpp:765-779` → `localsocketcontroller.cpp:257` → парсинг в `daemon.cpp:406`. Реестр бокса на 23.10: `Conf/appsRouteMode=2`, `Conf/appsSplitTunnelingEnabled=true`, `Conf/ExceptApps` = iexplore.exe + curl.exe — данные на месте
- [ ] только после успешного теста — коммитить правки `client/platforms/windows/daemon/` в `dev` (сейчас они в рабочем дереве и собраны без коммита, см. `deploy/BUILD_RELEASE.md` §4.1)
- [ ] закоммитить новые скрипты хоста: `deploy/setup_windows_build.ps1`, `deploy/setup_windows_admin.ps1`, `deploy/run_windows_admin_task.ps1`
- [ ] отдать юзеру инструкцию: исключения работают только для приложений, запущенных ПОСЛЕ включения туннеля (сокет, открытый до engage, драйвер не умеет перевесить) — Steam/игру перезапускать после Connect.
- [ ] установить `deploy/build-macos-local/pkg/Dopamine-local-4.8.14.70.pkg` и погонять переподключения: в логе не должно быть серий «Already connected» / «Deactivation in progress» длиннее пары строк.
- [ ] проверить импорт шар-подписки: api 0.6.62 + mrkting 0.2.42 выкачены на прод 22.10 00:31/00:44, блокер снят.
- [ ] при необходимости Chrome fingerprint на XHTTP: вынести `fp` из REALITY-ветки и добавить в XHTTP-ссылки. Для совместимости не требуется.

## Blockers / Notes
- Android TV: APK требуют Android 9+. Тестовая подпись требует чистой установки при наличии официальной версии. ARMv7 проверен сборкой и подписью, AVD поддерживает только ARM64.
- **Блокер прогона:** бокс `winbuild` (192.168.1.18, ssh-алиас `winbuild`, ключ `~/Desktop/dopamine-win-bootstrap/dopamine_win_build`) после reboot на экране логина, AutoAdminLogon выключен — GUI-тест (запуск `Dopamine.exe`, клик «Подключиться») без логина юзера невозможен. SSH и админские задачи (SYSTEM через schtasks) работают
- Установленный MSI: `C:\build\dopamine\Dopamine_x64.msi` на боксе и копия `/Users/2pizza/Desktop/Dopamine_x64.msi` (собрано 23.10 16:44 из рабочего дерева)
- Секреты (`.p8`, `agw_private.pem`) не коммитить.

## Related
- `AGENTS.md`
