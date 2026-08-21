# ReactOS Build Farm

Repositorio auxiliar para ejecutar builds dirigidos de ReactOS en GitHub Actions sin cargar CPU, RAM ni disco de DASHF15.

## Principio

La compilación remota es una opción adicional, no un reemplazo destructivo del pipeline local. Los worktrees, RosBE, CMake/Ninja, QEMU y los procedimientos locales existentes se conservan.

El workflow `ReactOS Target Build` recibe un repositorio público ReactOS, una rama/tag/SHA y una lista de targets. Configura RosBE en un runner Linux, compila sólo los targets solicitados y publica logs + artefactos encontrados. Opcionalmente también arranca un ISO en QEMU TCG headless, captura COM1/COM2 y exige un marker de runtime antes de declarar éxito.

Defaults deliberados para trabajo iterativo:

- `alesanGreat/reactos`
- `i386`
- GCC
- Debug
- `ENABLE_ROSTESTS=1`
- `I18N_LANG=en-US`

## Uso recomendado desde el workspace ReactOS

El helper canónico es:

`C:\Team Dropbox\Valis Idealis\Ale\Programacion\ReactOS\tooling\reactos-actions.py`

Ejemplo para una rama ya pusheada al fork:

```cmd
C:\Users\alesanGreat\miniforge3\python.exe "C:\Team Dropbox\Valis Idealis\Ale\Programacion\ReactOS\tooling\reactos-actions.py" run rostests-354-wininet-timeouts wininet_winetest
```

Ejemplo de build + runtime QEMU totalmente remoto:

```cmd
C:\Users\alesanGreat\miniforge3\python.exe "C:\Team Dropbox\Valis Idealis\Ale\Programacion\ReactOS\tooling\reactos-actions.py" run SHA bootcd --runtime-qemu --runtime-iso bootcd.iso --runtime-marker RAMDISK_PROBE_DONE --runtime-timeout-seconds 300
```

Consultar, esperar y descargar artefactos:

```cmd
C:\Users\alesanGreat\miniforge3\python.exe "C:\Team Dropbox\Valis Idealis\Ale\Programacion\ReactOS\tooling\reactos-actions.py" status RUN_ID
C:\Users\alesanGreat\miniforge3\python.exe "C:\Team Dropbox\Valis Idealis\Ale\Programacion\ReactOS\tooling\reactos-actions.py" wait RUN_ID
C:\Users\alesanGreat\miniforge3\python.exe "C:\Team Dropbox\Valis Idealis\Ale\Programacion\ReactOS\tooling\reactos-actions.py" download RUN_ID --destination C:\SourceCode\SistemasOperativos\ReactOS\github-actions-artifacts\RUN_ID --extract
```

El helper reutiliza Git Credential Manager sin imprimir el token, autentica las consultas para evitar el rate limit anónimo, tolera fallos transitorios de lectura y protege la credencial al seguir redirects de artifacts. GitHub CLI sigue siendo una alternativa válida, pero el helper evita depender de los bloqueos intermitentes observados en algunos subcomandos `gh api`/`gh repo` de DASHF15.

## Reglas operativas

1. GitHub Actions sólo puede compilar lo que exista en GitHub. Cambios locales sin commit/push no están incluidos.
2. Para trabajo WIP valioso, hacer commit de trabajo y push al fork antes de disparar el build remoto. Los WIP commits pueden limpiarse/squashearse más tarde.
3. Preferir Actions para builds pesados, configuraciones desde cero, matrices, builds simultáneos y runtimes QEMU automatizables.
4. La carga de DASHF15 no decide si se usa el farm: si el oracle puede reproducirse remotamente, usar el farm aunque la laptop esté ociosa. Mantener local QEMU sólo cuando exista una dependencia técnica real del host/hardware.
5. No eliminar ni degradar `C:\RosBE-portable`, `C:\SourceCode\SistemasOperativos\ReactOS\build`, scripts locales o recetas de Ninja/CMake. Ambos caminos son soportados.
6. No usar esta granja para abrir PRs upstream automáticamente. Su función es build/QA; publicación upstream sigue el flujo ReactOS existente.
7. Los inputs `targets`, `source_repository` e `i18n_lang` se validan antes de interpolarse en comandos para evitar inyección accidental.
8. El runbook completo del workspace está en `docs/github-actions-builds.md`; la política es remoto primero para carga pesada y local preservado como opción/fallback.
9. El bootstrap de RosBE valida que `RosBE-CI/RosBE.sh` exista realmente antes de continuar y reintenta descargas transitorias (incluidos HTTP 5xx). Un script de bootstrap que termine con código 0 pero deje un toolchain incompleto se considera fallo, nunca build exitoso.
10. Ninja se instala por `apt` sólo si realmente falta. `ccache` es una optimización opcional: si el runner no lo trae, el build continúa con `ENABLE_CCACHE=0` en vez de quedar bloqueado descargando paquetes. No reintroducir un `apt-get update/install` obligatorio en cada build.

## Artefactos

Cada run conserva durante 14 días:

- comando exacto de configure;
- log de configure;
- comando exacto de build;
- log de build;
- SHA real compilado;
- estadísticas de ccache;
- binarios/ISOs cuyo nombre coincida con los targets solicitados cuando puedan localizarse automáticamente;
- si `runtime_qemu` está habilitado: `runtime/qemu-command.txt`, `runtime/qemu.log`, `runtime/com1.log`, `runtime/com2.log` y `runtime/result.txt`.

La ausencia de un binario en el artifact no implica que el target no haya compilado; el `build.log` y el exit status del job son la evidencia primaria.
