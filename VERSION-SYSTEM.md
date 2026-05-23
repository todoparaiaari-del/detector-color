# Sistema de Versionado - ChromaLens

## Características
- ✅ La versión actual se muestra en la pantalla de selección de gama (arriba a la derecha)
- ✅ Cada cambio incrementa automáticamente la versión
- ✅ Los respaldos se guardan en `_old/` con el sufijo de versión

## Cómo usar

### Paso 1: Antes de editar el HTML
Cuando vayas a hacer cambios en el `index.html`, ejecuta desde la carpeta `scripts/`:

```bash
./increment-version.bat
```

O si prefieres en PowerShell:
```powershell
cd scripts
powershell -ExecutionPolicy Bypass -File ".\increment-version.ps1"
```

O desde la carpeta raíz del proyecto:
```powershell
powershell -ExecutionPolicy Bypass -File ".\scripts\increment-version.ps1"
```

### Paso 2: El script automáticamente:
1. ✅ Copia el `index.html` actual a `_old/index-vXXX.html` 
2. ✅ Incrementa la versión (0.31 → 0.32 → 0.33, etc.)
3. ✅ Actualiza la variable `VERSION` en el HTML
4. ✅ Actualiza el número de versión que se muestra en pantalla

### Ejemplo
```
Antes: v0.31
Script: ./scripts/increment-version.bat
Después: v0.32 (listo para editar)
Respaldo: _old/index-v031.html
```

## Versiones almacenadas
- `_old/index-v030.html` - Versión 0.30 respaldada
- `_old/index-v031.html` - Versión 0.31 respaldada
- `_old/index-v032.html` - Versión 0.32 respaldada
- Y así sucesivamente...

## ¿Dónde aparece la versión?
La versión aparece en la esquina superior derecha de la pantalla de selección de gama de colores, en gris claro (en los 3 idiomas).

## Archivos
- `scripts/increment-version.bat` - Script batch para Windows (doble clic para ejecutar)
- `scripts/increment-version.ps1` - Script PowerShell (multiplataforma)

