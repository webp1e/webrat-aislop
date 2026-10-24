# WebRAT / Salat Stealer — полный статический отчёт

> **Образец живой.** Ни `infected.exe`, ни извлечённые PE не запускались. Всё ниже получено чтением файлов, дизассемблированием, разбором Go pclntab, PyInstaller CArchive и расшифровкой конфигурации.

## 1. Метаданные

| Поле | Значение |
|---|---|
| Имя | `infected.exe` |
| Размер | 11 137 160 байт |
| SHA-256 | `a5481e564046c6e27c76121ac90b7e3acaf63958bbd22a01bcb7398ba24ea863` |
| Формат | PE32+ x86-64, GUI, 7 секций |
| Упаковщик/сборщик | PyInstaller, Python 3.11 |
| Семейство | Salat Stealer / WebRAT (Go RAT + infostealer) |

Внутри PyInstaller CArchive найдены два вложенных PE и Python-загрузчик:

| Файл | Размер | SHA-256 | Роль |
|---|---:|---|---|
| `exe1.exe` | 3 594 240 | `6b111dc2230cddeb61c7307d307b0ff59f60115242ef6b218073f626175c6337` | основной payload, UPX, Go 1.24 i386 |
| `exe1_unpacked.exe` | 12 572 160 | `eb4b1e1158621224572e1543a94cc195ffbc9abc76bc28bab0e166f71db0862e` | распакованный Go-бинарь |
| `exe2.exe` | 1 064 690 | `cc77f7a00e6f2d7ff2c77096cda81432aa9eff36a58dc66ccda8f59a0758db25` | фальшивый Telegram-прокси, decoy |

## 2. Цепочка запуска

Главный скрипт PyInstaller — `_payload_temp.pyc` (`_payload_temp.py`, Python 3.11). В нём два огромных base64-константы:

- `EXE1_B64` — 4 792 320 символов → 3 594 240 байт;
- `EXE2_B64` — 1 419 588 символов → 1 064 690 байт.

Функция `run_exe_from_b64`:

1. `base64.b64decode`;
2. каталог `%TEMP%` (или `tempfile.gettempdir()`);
3. имя `tmp_<uuid4().hex[:8]>.exe`;
4. запись файла;
5. `subprocess.run([path], shell=True, creationflags=0x08000000)` — `CREATE_NO_WINDOW`;
6. при `wait=True` ждёт завершения.

В `__main__` вызовы идут последовательно:

```python
path1 = run_exe_from_b64(EXE1_B64, wait=True)
path2 = run_exe_from_b64(EXE2_B64, wait=True)
```

То есть сначала работает стилер, потом запускается «Telegram Proxy». Исключения глушатся, дропнутые файлы не удаляются.

## 3. `exe2.exe` — декоративный Telegram-прокси

`exe2.exe` — PE32+ x86-64, собран MinGW, исходник `telegram_proxy.cpp`. В импортах только:

```text
d2d1.dll, DWrite.dll, GDI32.dll, USER32.dll, KERNEL32.dll, msvcrt.dll
```

Нет `ws2_32`, `wininet`, `winhttp`, `urlmon`, `crypt32`, `advapi32`, `shell32`; нет delay-import, экспортов и сетевых строк. Это не прокси и не дроппер.

Что делает:

- регистрирует класс окна `TGP`, создаёт окно `Telegram Proxy` 700×520, `WS_POPUP | WS_EX_LAYERED`;
- рисует интерфейс через Direct2D/DirectWrite;
- по клику на «Connect» запускает таймер 33 мс и увеличивает прогресс на `0.016` за тик;
- примерно через 2.1 секунды флаги переключаются в «connected»;
- «Ping» и частицы — косметика;
- в ресурсах — одна иконка 256×256 и манифест `asInvoker`.

В `.rdata` есть UTF-16-строки `proxy.telegram.org`, `Port`, `443`, `Secret`, `MTProto 2.0`, `Telegram Proxy v2.1.0 · End-to-end encrypted`, но «secret» — 5 байт, а не валидный 16-байтовый MTProto-secret. Сокетов нет.

Вывод: `exe2` — социальная маскировка и отвлекалка. Реальной сетевой функции не несёт.

## 4. `exe1.exe` — основной payload

После `upx -d`:

```text
PE32 executable, Intel i386, Go 1.24.0
path  salat
mod   salat (devel)
```

Go build info:

```text
CGO_ENABLED=0
GOARCH=386
GOOS=windows
GO386=sse2
```

Исходные файлы, сохранившиеся в pclntab:

```text
salat/funcs.go
salat/init.go
salat/main.go
salat/sets.go
salat/task.go
salat/tsc.go
```

Ключевые зависимости:

```text
github.com/gorilla/websocket
github.com/quic-go/quic-go
github.com/tetratelabs/wazero
github.com/ncruces/go-sqlite3
github.com/xssnick/tonutils-go
github.com/capnspacehook/taskmaster
github.com/buger/jsonparser
golang.org/x/crypto
```

Символы `main.*` дают почти полную карту возможностей: `c2Server`, `initConnection`, `changeEndpoint`, `recvWss`, `execCommand`, `shellCommand`, `downloadFile`, `screenStream`, `ffdesktop`, `ffwcam`, `ffwmic`, `runKeylogger`, `p2pSocks`, `proxySocks`, `Steal`, `getChrome*`, `getGecko*`, `getDiscord`, `getSteams`, `DPAPI`, `impersonateSystem`, `staticinstall`, `Suicide`, `newTask`.

## 5. Криптография

### 5.1 AES-ключ конфигурации

В `main.main` по адресу `0x9dd18f` лежит 4-байтовый сид:

```text
"biba"
```

Далее:

```text
MD5("biba") = 931691969b142b3a0f11a03e36fcc3b7
```

К MD5 применяется forward cumulative XOR:

```text
key[0] = md5[0]
key[i] = md5[i] ^ key[i-1]   для i = 1..15
```

Результат:

```text
AES-128 key = 93851482190d261c1302a29caa569522
```

Ключ складывается в глобальный слайс `0xfaee28` и используется в `main.dec` (режим 1) как ключ AES-GCM.

### 5.2 Проверка ключа

В `main.GetHWID` (`0x8d2a90`) собирается 54-байтовый блок:

```text
d8 f9 03 40 80 1d f0 03 46 59 14 43
fe 64 2a 9a 05 dd 06 df bb 9c 42 ef
d3 5d ea 39 a0 4d df 26 10 8c bc 31
ac 5f f9 8a 2f 24 44 ef 57 a6 c8 3c
72 fc 94 36 c7 17
```

Первые 12 байт — nonce:

```text
d8f90340801df00346591443
```

Остальное — ciphertext + GCM tag. Расшифровка ключом выше даёт 26-байтовую UTF-8 строку:

```text
"Я люблю сосать"
```

GCM-тег сходится — ключ подтверждён криптографически, а не только по коду.

### 5.3 Вторичный XOR-ключ

Расшифрованная строка (26 байт) сохраняется в `0xfaee38` и используется как ключ повторяющегося XOR в `main.dec`:

| Режим | Операция |
|---|---|
| 1 | AES-128-GCM Open: nonce (12) + ct+tag |
| 2 | forward cumulative XOR: `b[i] ^= b[i-1]` |
| 3 | repeating-key XOR с ключом `"Я люблю сосать"` |
| 4 | mode 2 → mode 3 |
| 5 | mode 3 → mode 2 |

Режимы 4/5 используются для строк/JSON-фреймов; режим 1 — для бинарного конфига.

### 5.4 RSA-слой конфига

В `.rdata` лежат девять 516-байтовых блобов с заголовком `a5 a7 a5 a5`:

```text
0xC94278  modulus (4096-bit)
0xC9447C  config 1
0xC94680  config 2
0xC94884  config 3
0xC94A88  config 4
0xC94C8C  config 5
0xC94E90  config 6
0xC95094  config 7
0xC95298  config 8
0xC9549C  config 9
```

`main.gb` (`0x8dfb60`):

1. проверяет `dword0 ^ 0xa5a5a5a5 == 0x200`;
2. читает dwords 1..128;
3. для `i=0..127`: `ror32(w, 4*i)`, затем `^ 4*i`, затем `^ 0x9e3779b9`;
4. делает `bswap` и собирает big-endian 512-байтовое число.

Далее для каждого конфига:

```text
H1 = pow(gb(blob), 65537, modulus)
B1 = hex_decode(H1)
B2 = repeating_xor(cumulative_xor(B1), "Я люблю сосать")
B3 = hex_decode(B2)
plaintext = AES-GCM(key, B3[:12], B3[12:])
```

То есть конфиг защищён тремя слоями: **собственное кодирование → RSA → XOR → AES-GCM**. Расшифровка подтверждена GCM-тегами для всех девяти записей.

## 6. C2 и конфигурация

| № | Адрес блоба | Расшифрованное значение |
|---:|---|---|
| 1 | `0xC9447C` | `https://salator.es/sa1at/` |
| 2 | `0xC94680` | `https://wruser.org/sa1at/` |
| 3 | `0xC94884` | `https://websalat.top/sa1at/` |
| 4 | `0xC94A88` | `https://sa1atik.cn/sa1at/` |
| 5 | `0xC94C8C` | `https://wruser.org:992/sa1at/` |
| 6 | `0xC94E90` | `https://tonapi.io/v2/dns/webratpanel.ton/resolve` |
| 7 | `0xC95094` | `https://tonapi.io/v2/dns/remoteadministratortool.ton/resolve` |
| 8 | `0xC95298` | `shyetootval.ton` |
| 9 | `0xC9549C` | `webratxaye1.ton` |

`tonapi.io` — легитимный публичный TON API. Вредоносны TON-имена:

```text
webratpanel.ton
remoteadministratortool.ton
shyetootval.ton
webratxaye1.ton
```

Дополнительно в коде есть DoH-резолверы:

```text
https://dns.google/resolve?name=
https://cloudflare-dns.com/dns-query?name=
```

и `main.tonResolve` / `tryTonResolve` / `getBC` / `decodeFromTonAddress` — то есть адрес C2 может обновляться через TON DNS / on-chain данные, а не только из статического конфига.

## 7. Возможности

По символам, строкам и импортам (не по динамическому поведению):

### Стилер

- Chromium-семейство: Chrome, Edge, Brave, Opera, Yandex, Sputnik, 360, Elements, DCBrowser, K-Meleon и др.
- Chrome `Local State`, `local_encryptor_data`, `app_bound_encrypted_key`, cookies, logins, autofill, tokens.
- Gecko/Firefox/SeaMonkey: `logins.json`, `key4.db`, `cookies.sqlite`, `nssPrivate`, `moz_cookies`.
- Discord tokens (`Clients\DiscordTokens.txt`).
- Steam (`SOFTWARE\Valve\Steam`, `loginusers.vdf`, `ssfn*`).
- Telegram `tdata`.
- Крипто-кошельки: Exodus, Electrum, Atomic, Jaxx, Trust, MetaMask, TonKeeper, Phantom, Solflare, Keplr и др.
- DPAPI, `DuplicateUserTokenFromSessionID`, `ImpersonateLoggedOnUser`, попытки SYSTEM.

### RAT

- Выполнение shell-команд.
- Загрузка/выгрузка файлов.
- SOCKS5/P2P-прокси.
- Стрим экрана, вебкамера, микрофон.
- Кейлоггер, буфер обмена, активное окно.
- Управление процессами, привилегиями, завершение/self-delete.

### Сеть

- WebSocket (`gorilla/websocket`) — основной канал `wsSess`.
- QUIC/HTTP3 (`quic-go`).
- HTTP/HTTPS, DoH.
- TON DNS / TON API.

### Персистентность

- Task Scheduler через `github.com/capnspacehook/taskmaster` (`newTask`, `staticinstall`).
- Registry Run (`SOFTWARE\Microsoft\Windows\CurrentVersion\Run`).
- Маскировочные имена в строках: `winws.exe`, `bibawinv` (конкретные имена задач/значений динамикой не подтверждены).
- `Suicide` / self-delete.

## 8. Anti-analysis / evasion

- PyInstaller-обёртка + base64-вложенные PE.
- UPX на основном payload.
- Последовательный запуск: сначала стилер, потом decoy-прокси.
- `CREATE_NO_WINDOW`.
- Конфиг зашифрован RSA + AES-GCM + XOR; C2 не лежит plaintext.
- TON DNS/DoH для устойчивости к блокировкам.
- Подмена/маскировка под Telegram-прокси.
- Попытки повышения привилегий и impersonation.

## 9. IOC

### Файлы

```text
a5481e564046c6e27c76121ac90b7e3acaf63958bbd22a01bcb7398ba24ea863  infected.exe
6b111dc2230cddeb61c7307d307b0ff59f60115242ef6b218073f626175c6337  exe1.exe
eb4b1e1158621224572e1543a94cc195ffbc9abc76bc28bab0e166f71db0862e  exe1_unpacked.exe
cc77f7a00e6f2d7ff2c77096cda81432aa9eff36a58dc66ccda8f59a0758db25  exe2.exe
```

### Сеть

```text
salator.es
wruser.org
websalat.top
sa1atik.cn
webratpanel.ton
remoteadministratortool.ton
shyetootval.ton
webratxaye1.ton
```

URL:

```text
https://salator.es/sa1at/
https://wruser.org/sa1at/
https://websalat.top/sa1at/
https://sa1atik.cn/sa1at/
https://wruser.org:992/sa1at/
https://tonapi.io/v2/dns/webratpanel.ton/resolve
https://tonapi.io/v2/dns/remoteadministratortool.ton/resolve
```

### Поведение

- `%TEMP%\tmp_<8 hex>.exe`;
- запуск двух PE подряд с `CREATE_NO_WINDOW`;
- окно `Telegram Proxy` / класс `TGP` (decoy);
- процессы/имена `winws.exe`, `bibawinv` (строки);
- Task Scheduler + Run key;
- запросы к `tonapi.io` с именами `*.ton`;
- большой Go-бинарь с модулем `salat` (после снятия UPX).

### Detection notes

- Блокировать конкретные TON-имена и HTTPS C2; не блокировать `tonapi.io` целиком без необходимости.
- Искать `tmp_*.exe` в `%TEMP%`, порождённые PyInstaller-процессом.
- Проверять Task Scheduler и Run key на записи, совпадающие с `winws.exe` / `bibawinv`.
- В памяти искать AES-ключ `93851482190d261c1302a29caa569522` и строку `"Я люблю сосать"` — это устойчивые артефакты этой сборки.
- Для сетевого детекта полезны TLS/SNI к `salator.es`, `wruser.org`, `websalat.top`, `sa1atik.cn`, а также DNS-запросы к `*.ton` через `tonapi.io`.

## 10. Ограничения

- Образец не запускался: поведение восстановлено статически.
- Полный WebSocket-командный протокол (`recvWss`, десятки веток) не разобран построчно.
- Точные имена scheduled task/registry value не подтверждены динамикой.
- C2 могут ротироваться через TON; список актуален для этой сборки.
- AES-ключ зависит от сида `"biba"`; в другой сборке сид/ключ могут отличаться.
- Часть возможностей (например, конкретные браузеры/кошельки) выведена из строк и символов, а не из выполнения.

## 11. Артефакты и воспроизведение

Скрипты:

```bash
python3 scripts/derive_key.py
python3 scripts/decrypt_c2.py /path/to/exe1_unpacked.exe
```

Артефакты текущего разбора (вне репозитория, временные):

```text
/tmp/opencode/webrat-extract/
├── infected.exe_extracted/
├── payloads/exe1.exe
├── payloads/exe1_unpacked.exe
├── payloads/exe2.exe
└── work/verify_c2.py
```

Ключевые адреса (Rizin VA; для кода objdump показывает на `0x400000` меньше):

| Что | Адрес |
|---|---|
| сид `"biba"` | `0x9dd18f` |
| `main.dec` | `0x8d3170` |
| `main.gb` | `0x8dfb60` |
| `main.GetHWID` | `0x8d2a90` |
| глобальный AES-ключ | `0xfaee28` |
| вторичный XOR-ключ | `0xfaee38` |
| RSA-модуль | `0xc94278` |
| первый C2-блоб | `0xc9447c` |

## 12. Итог

Это не «просто стилер» и не «просто RAT», а сборка Salat/WebRAT с:

- многослойной конфигурацией (custom encoding + RSA + XOR + AES-GCM);
- устойчивым C2 через HTTPS, WebSocket, QUIC и TON DNS;
- широким стилером браузеров, кошельков, Telegram и Steam;
- RAT-функциями (shell, экран, камера, микрофон, кейлоггер, прокси);
- persistence и попытками повышения привилегий;
- decoy-приложением «Telegram Proxy» для маскировки.

Ключевой практический результат: **AES-128 ключ `93851482190d261c1302a29caa569522`** и полный список C2, восстановленный из RSA-конфига.
