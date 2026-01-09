# Taxo EC - Documentación de API

Este repositorio contiene la documentación estática de la API de Taxo EC, generada con [Scribe](https://scribe.knuckles.wtf/).

## 🚀 Despliegue

Esta documentación se despliega automáticamente en **Cloudflare Pages** desde la rama `main`.

**URL de producción:** [Tu URL de Cloudflare]

## 🔄 Workflow para Actualizar la Documentación

### 1️⃣ Generar Documentación (desde el repo principal)

```bash
# En /home/isidro/code/taxo-ec (repositorio principal)
sail artisan scribe:generate
```

✅ En este punto, la documentación funciona correctamente en **local** en `http://localhost/docs`

### 2️⃣ Preparar para Cloudflare

```bash
# Desde el repositorio principal
cd /home/isidro/code/taxo-ec
./fix-docs-paths.sh fix
```

Este script:
- ✅ Corrige las rutas de assets para Cloudflare (`../docs/css/` → `css/`)
- ✅ Crea un backup automático (`index.html.backup`)

### 3️⃣ Commitear y Desplegar

```bash
# Entrar al submódulo
cd public/docs

# Verificar cambios
git status

# Agregar cambios
git add .

# Commitear
git commit -m "docs: Update API documentation with new endpoints"

# Push a Cloudflare
git push origin main
```

⏱️ **Cloudflare desplegará automáticamente** en 1-2 minutos.

### 4️⃣ Actualizar Referencia en Repo Principal

```bash
# Volver al repo principal
cd ../..

# Agregar la nueva referencia del submódulo
git add public/docs

# Commit
git commit -m "chore: Update docs submodule reference"

# Push
git push origin develop
```

### 5️⃣ (Opcional) Revertir para Ver Localmente

Si necesitas ver la documentación localmente después de corregir las rutas:

```bash
# Desde el repositorio principal
./fix-docs-paths.sh revert
```

Esto restaura las rutas para visualización local (`css/` → `../docs/css/`).

## 🔧 Diferencias Local vs Cloudflare

| Entorno | URL Base | Rutas de Assets | Funciona con |
|---------|----------|-----------------|--------------|
| **Local (Laravel)** | `/docs` | `../docs/css/...` | Scribe default |
| **Cloudflare Pages** | `/` | `css/...` | Después del script |

## 📝 Notas Importantes

1. **NUNCA** corras `./fix-docs-paths.sh fix` si quieres ver la documentación en local
2. El archivo `index.html.backup` está en `.gitignore` y no se commitea
3. Scribe siempre genera con rutas para Laravel, por eso necesitamos el script
4. El script puede revertir cambios si tienes el backup

## 🆘 Solución de Problemas

### La documentación no se ve en local

```bash
# Regenerar con Scribe
sail artisan scribe:generate
```

### La documentación no tiene estilos en Cloudflare

```bash
# Corregir rutas y hacer push
./fix-docs-paths.sh fix
cd public/docs
git add index.html
git commit -m "fix: Correct asset paths for Cloudflare"
git push origin main
```

### Perdí el backup

No hay problema, simplemente regenera con Scribe:

```bash
sail artisan scribe:generate
```

## 📚 Recursos

- [Scribe Documentation](https://scribe.knuckles.wtf/)
- [Cloudflare Pages Docs](https://developers.cloudflare.com/pages/)
- Repositorio principal: [Taxo-App/taxo-ec](https://github.com/Taxo-App/taxo-ec)
