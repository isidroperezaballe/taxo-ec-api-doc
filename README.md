# TaxoEC API Documentation

![API Version](https://img.shields.io/badge/version-1.0.0-blue)
![License](https://img.shields.io/badge/license-Private-red)
![Status](https://img.shields.io/badge/status-production-success)

Documentación oficial de la API REST de TaxoEC para integración externa.

## 📖 Ver Documentación

La documentación está disponible en:

**🌐 https://docs.tudominio.com** _(una vez configurado Cloudflare Pages)_

## 📁 Contenido

Este repositorio contiene documentación estática generada automáticamente por [Scribe](https://scribe.knuckles.wtf/) que incluye:

- **📄 HTML Docs** - Documentación interactiva navegable
- **📮 Postman Collection** - `collection.json` para importar en Postman
- **📝 OpenAPI Spec** - `openapi.yaml` para Swagger/otras herramientas

## 🔗 Formatos Disponibles

### Documentación Web
Abre `index.html` en tu navegador o visita la URL de Cloudflare Pages.

### Postman
1. Abre Postman
2. File → Import
3. Selecciona `collection.json`
4. Configura las variables:
   - `base_url`: URL de tu API
   - `x-api-key`: Tu API key
   - `x-organization-id`: Tu organization ID

### OpenAPI/Swagger
Importa `openapi.yaml` en:
- [Swagger Editor](https://editor.swagger.io/)
- [Stoplight Studio](https://stoplight.io/studio)
- Otras herramientas compatibles con OpenAPI 3.0

## 🚀 Endpoints Documentados

### Taxpayers (Contribuyentes)
- `GET /api/v1/taxpayers` - Listar contribuyentes
- `POST /api/v1/taxpayers` - Crear nuevo contribuyente
- `GET /api/v1/taxpayers/{taxNumber}` - Obtener contribuyente específico
- `GET /api/v1/taxpayers/{taxNumber}/categories` - Obtener categorías del contribuyente

## 🔐 Autenticación

Todos los endpoints requieren los siguientes headers:

```http
x-api-key: YOUR_API_KEY_HERE
x-organization-id: YOUR_ORGANIZATION_ID
Content-Type: application/json
Accept: application/json
```

## 📋 Ejemplo de Uso

```bash
curl --request GET \
  --url 'https://api.tudominio.com/api/v1/taxpayers?page=1&per_page=15' \
  --header 'x-api-key: YOUR_API_KEY_HERE' \
  --header 'x-organization-id: YOUR_ORGANIZATION_ID' \
  --header 'Accept: application/json'
```

## 🔄 Actualizaciones

Esta documentación se actualiza automáticamente cuando se realizan cambios en la API.

**Última generación:** _(ver commits de Git)_

## 📞 Soporte

Para obtener tu API key o reportar problemas:
- **Email:** soporte@tudominio.com
- **Repositorio principal:** (privado)

## ⚙️ Información Técnica

- **Generador:** Scribe v5.x
- **API Version:** v1
- **Base URL:** `https://api.tudominio.com`
- **Rate Limiting:** Configurado por organización

## 📜 Licencia

Esta documentación es propiedad privada de TaxoEC. 
No está permitida su redistribución sin autorización.

---

**Generado con ❤️ por [Scribe](https://scribe.knuckles.wtf/)**
