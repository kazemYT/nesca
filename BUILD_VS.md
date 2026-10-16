# Сборка в Visual Studio (Windows)

В репозитории лежат готовые `nesca.sln` + `nesca.vcxproj` (+ `nesca.vcxproj.filters`).
Расширение **Qt VS Tools не требуется**: `moc`/`uic`/`rcc` вызываются через
Custom Build Steps, генерируемые файлы складываются в
`<Platform>\<Configuration>\obj\GeneratedFiles\`.

## Что нужно поставить

| Зависимость | Откуда | Зачем |
|---|---|---|
| Visual Studio 2022 (toolset v143, Windows 10/11 SDK) | visualstudio.microsoft.com | компилятор MSVC |
| **Qt 5.15.2** для MSVC (`msvc2019_64` и/или `msvc2019`) | qt.io / архивы Qt | Core, Gui, Widgets, Multimedia. **Qt 6 не подойдёт** — код использует `QTextCodec`, удалённый из Qt 6 |
| libssh, libcurl, OpenSSL | vcpkg, см. ниже | `SSHAuth`, `libcurl`-запросы, `openssl/err.h` |

## Сторонние библиотеки (vcpkg)

Зависимости уже описаны в `vcpkg.json` (libssh, curl, openssl) — ничего вручную
собирать не надо:

```bat
git clone https://github.com/microsoft/vcpkg C:\vcpkg
C:\vcpkg\bootstrap-vcpkg.bat
setx VCPKG_ROOT C:\vcpkg
C:\vcpkg\vcpkg install --triplet x64-windows
:: для Win32 дополнительно:
C:\vcpkg\vcpkg install --triplet x86-windows
```

Проект сам подхватывает `$(VCPKG_ROOT)\installed\<triplet>\include|lib`
(если `VCPKG_ROOT` не задан — ищет `vcpkg_installed\<triplet>` рядом с `.vcxproj`).

## Сборка

1. Указать путь к Qt — одним из способов:
   - переменная окружения `QTDIR` (например `setx QTDIR C:\Qt\5.15.2\msvc2019_64`), или
   - `msbuild nesca.sln /p:QTDIR=C:\Qt\5.15.2\msvc2019_64`, или
   - поправить дефолт в начале `nesca.vcxproj` (по умолчанию именно этот путь).
2. Открыть `nesca.sln`, выбрать `Release | x64` (или `Win32`), Build Solution.

Командная строка:

```bat
msbuild nesca.sln /m /p:Configuration=Release /p:Platform=x64
```

Готовый `nesca.exe` ляжет в `x64\Release\`; после сборки автоматически
запускается `windeployqt`, который докладывает рядом Qt DLL и плагины.

## Нюансы

- **Hikvision**: лежащий в репозитории `HCNetSDK.lib` — только x86, поэтому в
  линковку он сознательно **не** включён: в рантайме код сам делает
  `LoadLibrary("HCNetSDK.dll")` + `GetProcAddress` (см. `enableHikvisionSupport()`),
  т.е. для работы фичи достаточно положить `HCNetSDK.dll` рядом с exe, а для сборки
  она не нужна вовсе — и x64-билд из-за неё не падает.
- Системные либы `ws2_32.lib` и `iphlpapi.lib` (вторая также подключена через
  `#pragma comment` в `Connector.h`) уже прописаны в проекте.
- Рабочая директория при отладке из VS выставлена в корень проекта — программа
  ищет рядом списки `login.txt`, `pass.txt`, `negatives.txt` и т.д.
- Код старый (Qt5/MinGW эпохи, `-fpermissive`), поэтому в проекте отключён
  `/permissive-` (ConformanceMode=false), стандарт — C++14, кодировка исходников — UTF-8.
