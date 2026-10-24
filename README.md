# webrat-aislop
разобранный webrat с помощью deepseek!!!

# WebRAT / Salat Stealer — статический разбор

> **Это живой malware.** Ничего из этой папки не нужно запускать. Разбор выполнялся только статически: без запуска `infected.exe`, без запуска вложенных PE и без обращения к C2.

Полный технический отчёт: [REPORT.md](REPORT.md)

## TL;DR

`infected.exe` — PyInstaller-обёртка (Python 3.11), которая из `%TEMP%` достаёт и последовательно запускает два PE:

- `exe1.exe` — UPX-упакованный Go 1.24 x86 RAT/стилер семейства **Salat / WebRAT**;
- `exe2.exe` — фальшивый Telegram MTProto-прокси, декоративное GUI без сетевого кода.

Внутри `exe1` восстановлены:

- **AES-128 ключ конфигурации:** `93851482190d261c1302a29caa569522`
- **вторичный XOR-ключ:** 26-байтовая UTF-8 строка (см. отчёт)
- **9 C2/конфигурационных записей**, включая прямые HTTPS-URL и TON-имена
- RSA-слой конфига: 4096-битный модуль, `e=65537`, девять 516-байтовых блобов

## Быстрые факты

| Параметр | Значение |
|---|---|
| Исходный файл | `infected.exe`, 11 137 160 байт |
| SHA-256 | `a5481e564046c6e27c76121ac90b7e3acaf63958bbd22a01bcb7398ba24ea863` |
| Упаковка | PyInstaller, Python 3.11 |
| Основной PE | `exe1.exe`, UPX, PE32 i386, Go 1.24 |
| SHA-256 `exe1.exe` | `6b111dc2230cddeb61c7307d307b0ff59f60115242ef6b218073f626175c6337` |
| SHA-256 `exe1_unpacked.exe` | `eb4b1e1158621224572e1543a94cc195ffbc9abc76bc28bab0e166f71db0862e` |
| Второй PE | `exe2.exe`, PE32+ x86-64, MinGW, decoy |
| SHA-256 `exe2.exe` | `cc77f7a00e6f2d7ff2c77096cda81432aa9eff36a58dc66ccda8f59a0758db25` |
| AES-128 key | `93851482190d261c1302a29caa569522` |
| C2 | 5 HTTPS URL + 4 TON-имени |

## Структура репозитория

```text
.
├── README.md                 этот файл
├── REPORT.md                 полный технический отчёт
└── scripts/
    ├── derive_key.py         вывод AES-ключа и проверка GCM-блока
    └── decrypt_c2.py         расшифровка RSA/AES-конфига из exe1_unpacked.exe
```

## Проверка ключа

```bash
python3 scripts/derive_key.py
```

Ожидаемый вывод:

```text
MD5("biba") = 931691969b142b3a0f11a03e36fcc3b7
AES-128 key = 93851482190d261c1302a29caa569522
secondary key (26 bytes) = ...
GCM tag: OK
```

Скрипт использует `cryptography`; при необходимости:

```bash
python3 -m venv .venv
. .venv/bin/activate
pip install cryptography
```

## Расшифровка C2

Нужен **распакованный** `exe1_unpacked.exe` (см. раздел «Воспроизведение»). Скрипт не запускает PE, только читает файл.

```bash
python3 scripts/decrypt_c2.py /path/to/exe1_unpacked.exe
```

Ожидаемый результат:

```text
https://salator.es/sa1at/
https://wruser.org/sa1at/
https://websalat.top/sa1at/
https://sa1atik.cn/sa1at/
https://wruser.org:992/sa1at/
https://tonapi.io/v2/dns/webratpanel.ton/resolve
https://tonapi.io/v2/dns/remoteadministratortool.ton/resolve
shyetootval.ton
webratxaye1.ton
```

`tonapi.io` — легитимный публичный API; вредоносны именно TON-имена `webratpanel.ton`, `remoteadministratortool.ton`, `shyetootval.ton`, `webratxaye1.ton`.

## Воспроизведение разбора

Все шаги статические. Не запускать полученные PE.

```bash
# 1. Разобрать PyInstaller-архив
pyinstxtractor-ng infected.exe

# 2. Снять UPX с основного payload
upx -d infected.exe_extracted/exe1.exe -o exe1_unpacked.exe

# 3. Проверить ключ
python3 scripts/derive_key.py

# 4. Расшифровать конфиг/C2
python3 scripts/decrypt_c2.py exe1_unpacked.exe
```

## IOC (кратко)

Хэши:

```text
a5481e564046c6e27c76121ac90b7e3acaf63958bbd22a01bcb7398ba24ea863  infected.exe
6b111dc2230cddeb61c7307d307b0ff59f60115242ef6b218073f626175c6337  exe1.exe
eb4b1e1158621224572e1543a94cc195ffbc9abc76bc28bab0e166f71db0862e  exe1_unpacked.exe
cc77f7a00e6f2d7ff2c77096cda81432aa9eff36a58dc66ccda8f59a0758db25  exe2.exe
```

Сеть:

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

Поведение:

- доставка через `%TEMP%\tmp_<8 hex>.exe`;
- последовательный запуск двух PE с `CREATE_NO_WINDOW`;
- маскировка под Telegram-прокси;
- persistence через Task Scheduler / registry Run (по символам и строкам);
- config/C2 расшифровываются только в памяти.

## Ограничения

Это статический разбор. Полный WebSocket-протокол, все ветки `recvWss`, точные имена scheduled task/registry value и поведение в рантайме не разбирались построчно. C2 могут ротироваться, а AES-ключ зависит от сида `"biba"`: при другой сборке ключ будет другим.

## Правовое

Материал предназначен для защитного анализа, incident response и threat intelligence. Не использовать для запуска, распространения или модификации malware.
