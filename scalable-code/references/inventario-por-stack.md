# Inventario por stack

Comandos que la regla deja escritos para que el inventario sea
determinista. Se reemplazan `{web}`, `{api}`, `{db}` y `{Prefijo}` por las
rutas y el prefijo reales (relativos a la raíz del repo) y **cada comando
se ejecuta antes de dejarlo en la regla**: debe listar los archivos
compartidos que ya existen.

## Listar lo compartido

```bash
# Angular: todo lo que ya existe en core/ y shared/
find {web}/src/app/core {web}/src/app/shared -name '*.ts' ! -name '*.spec.ts' | sort

# .NET: piezas compartidas de cada proyecto
find {api}/src \( -name bin -o -name obj \) -prune -o -type f -name '*.cs' \( \
  -path '*.Application/Common/*' -o -path '*.Application/Helpers/*' -o -path '*.Application/DTOs/Base/*' -o \
  -path '*.Application/Interfaces/Common/*' -o -path '*.Mapper/Resolvers/*' -o -path '*.Validator/Rules/*' -o \
  -path '*/{Prefijo}.Common/*' -o -path '*/{Prefijo}.Security/*' \) -print | sort

# SQL: functions y table types
find {db}/functions {db}/types -name '*.sql' 2>/dev/null | sort
```

## Buscar un texto o nombre concreto

```bash
grep -rn -F --include='*.ts'  '<texto o nombre>' {web}/src
grep -rn -F --include='*.cs'  '<texto o nombre>' {api}/src
grep -rn -F --include='*.sql' '<texto o nombre>' {db}
```

`-F` busca el texto literal (sin regex): sirve para mensajes, nombres de
métodos o fragmentos SQL. El número de resultados es el número de copias.

## Patrones que fallan (y su corrección)

| Falla | Por qué | Correcto |
|---|---|---|
| `-path '*/Application/Common/*'` | La carpeta es `{Prefijo}.Application/`, no `Application/` | `-path '*.Application/Common/*'` |
| `-path '*/Mapper/Resolvers/*'` | Igual: `{Prefijo}.Mapper/` | `-path '*.Mapper/Resolvers/*'` |
| `find` sin podar `bin`/`obj` | Lista copias compiladas y duplica resultados | `\( -name bin -o -name obj \) -prune -o …` |
| `FILES="a b"; prettier $FILES` en zsh | zsh no separa variables en palabras | escribir las rutas directamente |

Comprobación mínima tras armar los comandos: el de .NET debe listar al
menos `SpResult.cs` y `ApiResponse.cs`; el de Angular, los servicios de
`core/services/`. Si no aparecen, el patrón está mal.
