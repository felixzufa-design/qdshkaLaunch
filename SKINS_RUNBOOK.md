# Скины qdskins — быстрый runbook

Короткий порядок действий для добавления скина и выпуска новой версии мода.
Не нужно заново изучать проект — всё, что нужно, собрано здесь.

## Что где лежит

| Что | Где |
|---|---|
| Исходники мода | `dist/_mdk_tmp/skinmod/` (Gradle-проект, build/libs → jar) |
| Собрать мод | `cd dist/_mdk_tmp/skinmod && ./gradlew build --no-daemon` (без `--offline`: offline падает на `cacheVersionExecutableClient1.21.1`) |
| Версия мода | `gradle.properties` → `mod_version` |
| Конфиг адреса списка | `QdSkins.java` → `skinsUrl` = `https://felixzufa-design.github.io/skins/skins.json` |
| Сайт скинов | репозиторий `felixzufa-design/skins`, ветка `main`, Pages включены |
| Присланные скины | `skins/` в корне проекта (называть по нику: `Nick.png`) |
| Манифест | `manifest.json` и копия `dist/manifest.json` (держать одинаковыми; версия набора V1.4, НЕ путать с версией лаунчера) |
| Планы на будущее | `PLANS/README.md` (оглавление), `PLANS/*.md` |
| Релиз | `felixzufa-design/qdshkaLaunch`, тег V1.4 |
| Игра у игроков | `%APPDATA%\qDshnikLauncher` (mods/, logs/, crash-reports/) |

## Добавить скин (обычный случай, мод пересобирать НЕ надо)

1. PNG положить в `skins/Nick.png`, проверить: `PIL Image.open(...).size in {(64,64),(64,32)}`.
2. Добавить в `skins.json` (в `dist/_skinwork/skins/`) строку
   `"Nick": "https://felixzufa-design.github.io/skins/Nick.png"`, залить PNG и JSON
   в `felixzufa-design/skins` (Contents API PUT с base64; для существующего JSON — с `sha`).
3. Проверить: `curl -s -L -o f https://felixzufa-design.github.io/skins/Nick.png`
   и sha256 против локального, `curl` skins.json. Pages обновляется ~1 мин.
4. Сказать игрокам перезайти на сервер (список обновляется раз в ~5 мин, кэш на диске).

## Выпустить новую версию мода

1. Править код в `dist/_mdk_tmp/skinmod/src/main/java/ru/qdshnik/qdskins/`.
2. `mod_version++`, `./gradlew build --no-daemon`.
3. В jar gradle портит кириллицу в `META-INF/neoforge.mods.toml`
   (двойное перекодирование) — **пересобрать jar, подменив этот файл** своим
   UTF-8-текстом (образец в разделе «Шаблон toml»). Не добавлять BOM — NeoForge
   падает «Invalid bare key».
4. `size` + `sha256` нового jar → запись `mods/qdskins-X.Y.Z.jar` в `manifest.json`,
   удалив старую запись qdskins. Скопировать в `dist/manifest.json`.
5. `gh release upload V1.4 <jar> -R felixzufa-design/qdshkaLaunch`
   затем **пересоздать manifest.json в релизе**: старый файл удалить
   (`gh api -X DELETE .../releases/assets/<id>`), потом `gh release upload ... --clobber`.
   Просто `--clobber` без удаления НЕ обновляет CDN (releases/latest/download отдаёт старый).
6. Удалить старый `qdskins-*.jar` из релиза (`gh release delete-asset`), чтобы лаунчер
   не чистил его у игроков и манифест совпадал с релизом.
7. Проверить: sha256 живого `releases/latest/download/manifest.json` == локального;
   скачать jar по URL из манифеста, сверить size/sha256 и наличие
   `META-INF/neoforge.mods.toml`.

## Правила, чтобы не наступить на грабли

- В манифест класть только файлы `mods/...`; конфиги (например `config/qdskins-common.toml`)
  не класть, если файла нет в релизе — лаунчер сломается на 404.
- Лаунчер удаляет из `mods/` всё, чего нет в манифесте → в релизе не должно быть
  лишних jar-файлов, иначе `make_github_manifest.py` втянет их в раздачу.
- Готовый jar всегда дослан в `dist/ToGit/` (для gh-pages раздачи, там публичную часть).
- В toml писать кириллицу только UTF-8 без BOM. Проверка: `javap -v` на
  `classes/ru/qdshnik/qdskins/mixin/AbstractClientPlayerMixin.class` показывает `RETURN`.
- Пересборку exe лаунчера при смене jar делать НЕ надо — манифест читается с GitHub.

## Шаблоны команд

```bash
# собрать
cd dist/_mdk_tmp/skinmod && sed -i 's/^mod_version=.*/mod_version=X.Y.Z/' gradle.properties && ./gradlew build --no-daemon --console=plain | tail -3

# залить jar + пересоздать manifest в релизе (SHA = sha256sum dist/manifest.json)
gh release upload V1.4 jar -R felixzufa-design/qdshkaLaunch
gh release upload V1.4 dist/manifest.json -R felixzufa-design/qdshkaLaunch --clobber
RID=$(gh api repos/felixzufa-design/qdshkaLaunch/releases/tags/V1.4 --jq .id)
AID=$(gh api repos/felixzufa-design/qdshkaLaunch/releases/$RID/assets --jq '.[]|select(.name=="manifest.json")|.id')
gh api -X DELETE repos/felixzufa-design/qdshkaLaunch/releases/assets/$AID
gh release upload V1.4 dist/manifest.json -R felixzufa-design/qdshkaLaunch
```

## Шаблон toml (UTF-8, без BOM)

```toml
modLoader="javafml"
loaderVersion="[4,)"
license="All Rights Reserved"

[[mods]]
modId="qdskins"
version="X.Y.Z"
displayName="qDshnik Skins"
authors="qDshnik"
description='''Клиентские кастомные скины по нику для offline-сервера.'''

[[mixins]]
config="qdskins.mixins.json"

[[dependencies.qdskins]]
    modId="neoforge"
    type="required"
    versionRange="[21.1.0,)"
    ordering="NONE"
    side="CLIENT"

[[dependencies.qdskins]]
    modId="minecraft"
    type="required"
    versionRange="[1.21.1,1.22)"
    ordering="NONE"
    side="CLIENT"
```

## Формат skins.json (с 1.0.8)

Обе формы поддерживаются: `"Nick": "url.png"` и
`"Nick": {"skin": "url.png", "cape": "capes/Nick.png"}`. Поля `cape` → `PlayerSkin.capeTexture`,
`elytraTexture` остаётся `null` → ElytraLayer рисует плащ (как ваниль). Плащ качается
в `config/qdskins/cache/<ник>_cape.png`, кэш с 1.0.x не чистится.

## Приоритет скинов и диагностика

- Миксин `AbstractClientPlayerMixin` врезается в `RETURN` `getSkin()`: сван, если другой
  мод (Ronova и др.) уже подставил текстуру из своего namespace (`isModTexture`).
- Если «чужой мод перекрывает наш скин» — смотреть `%APPDATA%\qDshnikLauncher\logs\latest.log`
  и искать, какой мод в этот момент подставляет текстуру.
- «not a valid mod file» → битый `META-INF/neoforge.mods.toml` (BOM или парсинг) —
  пересобрать jar по шаблону выше.