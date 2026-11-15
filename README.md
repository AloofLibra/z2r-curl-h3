# z2r-curl-h3

Статические сборки **curl с HTTP/3** (OpenSSL native QUIC + nghttp3, musl) для
QUIC-медиапроверки профиля 5 в заторе: перебор UDP-стратегий с проверкой
googlevideo по настоящему HTTP/3 с самого роутера.

Бинарники собираются из **публичных исходников** без каких-либо модификаций
протокольного кода (см. `build.sh` — полный рецепт воспроизведения).

## Источники

| Компонент | Версия | Лицензия |
|---|---|---|
| curl | 8.16.0 (тег `curl-8_16_0`) | MIT |
| OpenSSL | 3.5.4 (LTS, native QUIC API) | Apache-2.0 |
| nghttp3 | 1.12.0 | MIT |
| musl + gcc | musl.cc cross toolchains | MIT / GPL (toolchain, в бинарник не попадает) |

См. `NOTICE`.

## Платформы

Имена `arch` = контракт затора (`z2r_quic_curl_arch`, ассет
`z2r-curl-h3-linux-<arch>.gz`). Пакуются UPX 5.2.1 (`--best --lzma`), где UPX
поддержал формат; не поддержал — бинарник едет без упаковки (на детект и
работу не влияет).

| arch | Тулчейн (musl.cc) | ABI | Примечание |
|---|---|---|---|
| `x86_64` | x86_64-linux-musl | — | |
| `i686` | i686-linux-musl | — | live: OpenWRT i686, h3 OK |
| `aarch64` | aarch64-linux-musl | — | |
| `arm` | arm-linux-musleabihf | armv6+/hf | `LIBS=-latomic` |
| `mipsel` | mipsel-linux-musl | soft-float, LE | Keenetic MT7621; `LIBS=-latomic`; live: MT7621, h3 OK |
| `mipssf` | mips-linux-musl | soft-float, BE | big-endian MIPS; `LIBS=-latomic` |
| `riscv64` | riscv64-linux-musl | — | OpenSSL-таргет `linux64-riscv64` |

Бинарники **полностью статические** (`make AM_LDFLAGS=-all-static`): работают
и на musl-системах (OpenWRT), и на Entware/uClibc (Keenetic) — интерпретатора
ld-musl не требуется. 32-битным arch без нативных 64-битных атомиков (arm,
mips) нужна `-latomic` на линковке, иначе curl configure не детектит OpenSSL
(`__atomic_*_8` в libcrypto).

MIPS — soft-float: статический бинарник без FPU-аби-привязки работает на
hard-float ядрах (MT7621 и пр.).

## Как ставится (автоматически, затором)

Ничего делать не нужно: `lib/quic_media.sh` затора при отсутствии h3-curl
предлагает скачать `z2r-curl-h3-linux-<arch>.gz` из rolling-релиза `latest`
этого репозитория (детект arch — по `uname -m` + ELF-байт для MIPS), проверяет
gzip и размер, ставит в `/opt/zator/tools/bin/curl-h3-<arch>`.
Порядок детекта: env `Z2R_QUIC_CURL` → `/opt/bin/curl` (если уже умеет HTTP/3)
→ наш установленный → `curl` из PATH.

## Ручная установка

```sh
arch=$(uname -m)   # mipsel/mipssf уточняйте по ELF-байту, см. затор
wget -qO /tmp/curl-h3.gz https://github.com/MarkinAlexander/z2r-curl-h3/releases/download/latest/z2r-curl-h3-linux-$arch.gz
gunzip -c /tmp/curl-h3.gz > /opt/zator/tools/bin/curl-h3-$arch && chmod 755 /opt/zator/tools/bin/curl-h3-$arch
```

Проверка: `curl-h3-$arch --version | grep -i http3` и, при наличии CA-бандла
(Entware: `CURL_CA_BUNDLE=/opt/etc/ssl/certs/ca-certificates.crt`, OpenWRT:
`/etc/ssl/certs/ca-certificates.crt`), любой `--http3-only` запрос.

## Сборка

```sh
sh build.sh <arch>    # x86_64 i686 aarch64 arm mipsel mipssf riscv64
```

Зависимости: wget, make, pkg-config, `upx` (опционально, иначе без упаковки).
Результат — `dist/z2r-curl-h3-linux-<arch>.gz`. CI собирает все платформы по
тегу (`v*` или workflow_dispatch с тегом) и обновляет rolling-релиз `latest` +
`sha256sums.txt`. Кросс-тулчейны musl.cc самохостятся релизом `toolchains-v1`
этого репозитория (musl.cc отказывает GitHub-раннерам), musl.cc — фолбэк.

Кэш сборки живёт в `.build/<arch>/` (перезапуск продолжает с места обрыва);
на чистую пересборку — `rm -rf .build`.

## Ограничения и заметки

- CA-бандл в рантайме через env `CURL_CA_BUNDLE`: компилированный путь curl на
  Entware/OpenWRT обычно не существует, а вызовы с `-q` игнорируют curlrc.
- Собрано минимально: без zlib/brotli/zstd/idn2/psl/nghttp2/ssh2 и без
  лишних протоколов — только HTTPS по h1.1/h2/h3, что и нужно медиапроверке.
- UPX-упакованный бинарник распаковывается в память при старте — на роутерах с
  малым ОЗУ это ~2.5 МиБ, меньше исходных 6.6 МиБ.
- mipsel/mipssf/riscv64/arm: live-проверка на соответствующем железе — по
  мере доступа; сборка детерминированно повторяет рецепт, но апгрейд версий
  требует нового тега и прогона CI.
