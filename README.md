# ReactOS Build Farm

Repositorio auxiliar para ejecutar builds dirigidos de ReactOS en GitHub Actions sin cargar CPU, RAM ni disco de DASHF15.

## Principio

La compilación remota es una opción adicional, no un reemplazo destructivo del pipeline local. Los worktrees, RosBE, CMake/Ninja, QEMU y los procedimientos locales existentes se conservan.

El workflow `ReactOS Target Build` recibe un repositorio público ReactOS, una rama/tag/SHA y una lista de targets. Configura RosBE en un runner Linux, compila sólo los targets solicitados y publica logs + artefactos encontrados.

Defaults deliberados para trabajo iterativo:

- `alesanGreat/reactos`
- `i386`
- GCC
- Debug
- `ENABLE_ROSTESTS=1`
- `I18N_LANG=en-US`

## Uso por GitHub CLI

Ejemplo para una rama ya pusheada al fork:

```cmd
gh workflow run build-reactos.yml --repo alesanGreat/reactos-build-farm --ref main -f source_repository=alesanGreat/reactos -f source_ref=rostests-354-wininet-timeouts -f arch=i386 -f compiler=gcc -f build_type=Debug -f targets="wininet_winetest" -F enable_rostests=true -F enable_rosapps=false -f i18n_lang=en-US
```

Luego consultar:

```cmd
gh run list --repo alesanGreat/reactos-build-farm --workflow build-reactos.yml --limit 10
gh run watch RUN_ID --repo alesanGreat/reactos-build-farm
gh run download RUN_ID --repo alesanGreat/reactos-build-farm
```

## Reglas operativas

1. GitHub Actions sólo puede compilar lo que exista en GitHub. Cambios locales sin commit/push no están incluidos.
2. Para trabajo WIP valioso, hacer commit de trabajo y push al fork antes de disparar el build remoto. Los WIP commits pueden limpiarse/squashearse más tarde.
3. Preferir Actions para builds pesados, configuraciones desde cero, matrices y builds simultáneos.
4. Mantener local para iteraciones pequeñas, pruebas que dependan de QEMU/hardware local, diagnóstico interactivo y fallback cuando Actions no esté disponible.
5. No eliminar ni degradar `C:\RosBE-portable`, `C:\ReactOS\build`, scripts locales o recetas de Ninja/CMake. Ambos caminos son soportados.
6. No usar esta granja para abrir PRs upstream automáticamente. Su función es build/QA; publicación upstream sigue el flujo ReactOS existente.
7. Los inputs `targets`, `source_repository` e `i18n_lang` se validan antes de interpolarse en comandos para evitar inyección accidental.

## Artefactos

Cada run conserva durante 14 días:

- comando exacto de configure;
- log de configure;
- comando exacto de build;
- log de build;
- SHA real compilado;
- estadísticas de ccache;
- binarios/ISOs cuyo nombre coincida con los targets solicitados cuando puedan localizarse automáticamente.

La ausencia de un binario en el artifact no implica que el target no haya compilado; el `build.log` y el exit status del job son la evidencia primaria.
