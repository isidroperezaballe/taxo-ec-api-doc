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

### 5️⃣ ✅ Ver Documentación Localmente

**¡Ya no necesitas revertir!** La documentación funciona tanto en local como en Cloudflare con las mismas rutas.

```bash
# Simplemente visita
http://localhost/docs
```

Las rutas personalizadas en `routes/web.php` se encargan de servir los assets correctamente.

## 🔧 Configuración Unificada (Local y Cloudflare)

✅ **Ahora local y Cloudflare usan las MISMAS rutas!**

| Entorno | URL Base | Rutas de Assets | Requiere Script |
|---------|----------|-----------------|-----------------|
| **Local (Laravel)** | `/docs` | `css/...`, `js/...` | ✅ Sí (automático) |
| **Cloudflare Pages** | `/` | `css/...`, `js/...` | ✅ Sí (automático) |

**Cómo funciona:**
- Laravel tiene rutas especiales en `routes/web.php` que sirven `/docs/css/*` y `/docs/js/*`
- Estas rutas hacen que las rutas relativas (`css/...`) funcionen igual en local y producción
- **Ya no necesitas revertir cambios** para ver la documentación localmente

## 📝 Notas Importantes

1. ✅ **Ahora SÍ puedes** correr `./fix-docs-paths.sh fix` y ver la documentación en local
2. El archivo `index.html.backup` está en `.gitignore` y no se commitea
3. Scribe siempre genera con rutas `../docs/`, el script las corrige a rutas relativas `css/`, `js/`
4. Laravel tiene rutas especiales (`/docs/css/*`, `/docs/js/*`) que sirven los assets correctamente
5. **Las mismas rutas funcionan en local y en Cloudflare** 🎉

## 🆘 Solución de Problemas

### La documentación no se ve en local

```bash
# Regenerar con Scribe y corregir rutas
sail artisan scribe:generate
./fix-docs-paths.sh fix
```

Luego visita: http://localhost/docs

### La documentación no tiene estilos en local o Cloudflare

**Causa común:** Las rutas no se corrigieron después de regenerar.

```bash
# Corregir rutas
./fix-docs-paths.sh fix

# Para Cloudflare, además hacer push
cd public/docs
git add index.html
git commit -m "fix: Correct asset paths"
git push origin main
```

### Las rutas de Laravel no funcionan

Verifica que las rutas `/docs/css/*` y `/docs/js/*` estén registradas:

```bash
sail artisan route:list | grep docs
```

Deberías ver:
- `GET|HEAD  docs/css/{file}`
- `GET|HEAD  docs/js/{file}`
- `GET|HEAD  docs/{any?}`

## 📚 Recursos

- [Scribe Documentation](https://scribe.knuckles.wtf/)
- [Cloudflare Pages Docs](https://developers.cloudflare.com/pages/)
- Repositorio principal: [Taxo-App/taxo-ec](https://github.com/Taxo-App/taxo-ec)
