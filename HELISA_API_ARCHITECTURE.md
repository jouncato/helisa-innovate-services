# Arquitectura y API de Helisa

## Tabla de Contenidos
- [Descripción de la Arquitectura](#descripción-de-la-arquitectura)
- [API REST de Helisa](#api-rest-de-helisa)
- [Autenticación y Seguridad](#autenticación-y-seguridad)
- [Endpoints Disponibles](#endpoints-disponibles)
- [Modelos de Datos](#modelos-de-datos)
- [Manejo de Errores](#manejo-de-errores)

---

## Descripción de la Arquitectura

### Resumen General
Helisa implementa una **arquitectura de integración bidireccional** que conecta aplicaciones externas a través de una interfaz WebService utilizando comunicación JSON.

### Flujo Bidireccional

#### 1. Aplicación Externa → Helisa

El sistema sigue un proceso de 6 pasos:

1. **Inserción de datos** en la aplicación externa
2. **Lectura de datos no enviados** desde la aplicación externa
3. **Transmisión JSON** con firma de seguridad al WebService de Helisa
4. **Validación y almacenamiento** en la base de datos de Helisa
5. **Entrega de respuesta HTTP** con el resultado de la operación
6. **Marcado de registros** como enviados en caso de éxito

```
┌─────────────────┐                  ┌─────────────────┐
│  App Externa    │                  │     HELISA      │
│                 │                  │                 │
│  1. Inserción   │                  │                 │
│  2. Lectura     │                  │                 │
│  3. JSON+Sign   │───────────────>  │  4. Validación  │
│                 │                  │     + Insert DB │
│  6. Marcar      │  <───────────────│  5. HTTP Resp   │
│     enviado     │                  │                 │
└─────────────────┘                  └─────────────────┘
```

#### 2. Helisa → Aplicación Externa

Un proceso de recuperación en 4 pasos:

1. **Solicitud firmada** del cliente externo para obtener catálogos de datos
2. **Lectura y preparación** de información de catálogos por parte de Helisa
3. **Envío de respuesta JSON** con los datos solicitados
4. **Validación e inserción** local en la aplicación externa

```
┌─────────────────┐                  ┌─────────────────┐
│  App Externa    │                  │     HELISA      │
│                 │                  │                 │
│  1. Petición    │───────────────>  │  2. Lectura     │
│     firmada     │                  │     catálogos   │
│                 │                  │                 │
│  4. Validación  │  <───────────────│  3. JSON Resp   │
│     + Insert    │                  │                 │
└─────────────────┘                  └─────────────────┘
```

### Componentes Clave

| Componente | Descripción |
|------------|-------------|
| **Capa WebService** | Maneja validación, inserción de datos y generación de respuestas |
| **Formato JSON** | Protocolo estándar de intercambio de datos |
| **Capa de Base de Datos** | Almacena datos validados en ambos sistemas |
| **Mecanismo de Seguridad** | Firmas digitales verifican la autenticación del cliente para todas las transacciones |

### Características Técnicas

La arquitectura enfatiza la **sincronización e integridad de datos** a través de:

- ✅ Seguimiento de registros enviados vs. no enviados para prevenir duplicación
- 🔒 Firmas de seguridad obligatorias en todas las solicitudes
- 📊 Códigos de estado HTTP para retroalimentación de operaciones
- 📋 Esquemas de respuesta JSON estructurados

### Operaciones Soportadas

Esta arquitectura soporta múltiples operaciones incluyendo:

- Consultas de cuentas contables
- Gestión de inventario
- Inserción de documentos
- Listas de precios
- Reportes financieros

---

## API REST de Helisa

### Información General

Helisa proporciona una API REST a través del endpoint base `/KansasWS/` para integrar funcionalidades de contabilidad, inventario y gestión documental con su suite de software de contabilidad en la nube.

**Base URL**: `https://helisa.com/api/KansasWS/`

---

## Autenticación y Seguridad

### Método de Autenticación

Las solicitudes utilizan envío basado en formularios con tres parámetros principales:

| Parámetro | Tipo | Descripción |
|-----------|------|-------------|
| **id** | String | Identificador de la empresa |
| **json** | String/JSON | Payload de la solicitud (para peticiones POST) |
| **sign** | String | Firma de seguridad (hash HMAC de parámetros concatenados) |

### Generación de Firma de Seguridad

```javascript
// Pseudocódigo para generar la firma
const params = [id, json, secretKey].join('');
const sign = HMAC_SHA256(params);
```

**Importante**: La firma debe incluir todos los parámetros en orden específico para validación correcta.

---

## Endpoints Disponibles

### Endpoints de Consulta (GET)

#### 1. Cuentas Contables
**Endpoint**: `GET /KansasWS/accounts`

Recupera el plan de cuentas por tipo contable (local/NIIF).

**Parámetros**:
- `id`: Identificador de empresa
- `accountingType`: `local` | `NIIF`
- `sign`: Firma de seguridad

**Respuesta**:
```json
{
  "accounts": [
    {
      "code": "110505",
      "name": "Caja General",
      "nature": "D",
      "level": 3
    }
  ]
}
```

---

#### 2. Productos
**Endpoint**: `GET /KansasWS/products`

Obtiene inventario de productos, precios, familias y características.

**Información retornada**:
- Código y descripción del producto
- Precios de venta y compra
- Familias y subfamilias
- Características y atributos
- Estado activo/inactivo

---

#### 3. Centros de Costo
**Endpoint**: `GET /KansasWS/costCenters`

Códigos de asignación de costos organizacionales jerárquicos.

**Estructura**:
```json
{
  "costCenters": [
    {
      "code": "CC001",
      "name": "Administración",
      "level": 1,
      "parent": null
    }
  ]
}
```

---

#### 4. Terceros
**Endpoint**: `GET /KansasWS/thirdParties`

Datos de clientes, proveedores y terceros generales.

**Tipos de terceros**:
- Clientes
- Proveedores
- Empleados
- Otros

**Información incluida**:
- Identificación (NIT/CC)
- Nombre o razón social
- Dirección y contacto
- Régimen tributario
- Estado activo/inactivo

---

#### 5. Estados Financieros
**Endpoint**: `GET /KansasWS/financialStatements`

Balance general y estado de resultados por fecha.

**Parámetros**:
- `date`: Fecha de corte (formato: `{day, month, year}`)
- `statementType`: `balance` | `income`

**Respuesta**:
```json
{
  "statement": {
    "date": {"day": 31, "month": 12, "year": 2024},
    "type": "balance",
    "items": [
      {
        "account": "1105",
        "name": "Caja",
        "debit": 1000000,
        "credit": 0,
        "balance": 1000000
      }
    ]
  }
}
```

---

#### 6. Documentos
**Endpoint**: `GET /KansasWS/documents`

Resúmenes de transacciones filtrados por tipo y rango de fechas.

**Filtros disponibles**:
- Tipo de documento (factura, nota débito, nota crédito, etc.)
- Rango de fechas (desde/hasta)
- Estado (pendiente, aprobado, anulado)

---

#### 7. Cartera
**Endpoint**: `GET /KansasWS/portfolio`

Antigüedad de cuentas por cobrar a clientes y estado.

**Información retornada**:
- Cliente
- Documento
- Fecha de emisión
- Días de vencimiento
- Saldo pendiente
- Estado

---

#### 8. Movimientos de Inventario
**Endpoint**: `GET /KansasWS/inventoryMovement`

Transacciones de existencias entre bodegas por tipo de documento.

**Parámetros**:
- `warehouseId`: Identificador de bodega
- `documentType`: Tipo de movimiento
- `dateRange`: Rango de fechas

---

### Endpoints de Inserción/Actualización (POST)

#### 1. Comprobantes Contables
**Endpoint**: `POST /KansasWS/accountingEntry`

Inserta transacciones contables mixtas (débito/crédito).

**Estructura del payload**:
```json
{
  "id": "empresa123",
  "json": {
    "documentType": "CE",
    "date": {"day": 15, "month": 11, "year": 2024},
    "description": "Comprobante de egreso",
    "details": [
      {
        "account": "110505",
        "nature": "C",
        "value": 500000,
        "costCenter": "CC001",
        "description": "Pago servicios"
      },
      {
        "account": "510506",
        "nature": "D",
        "value": 500000,
        "costCenter": "CC001",
        "description": "Gasto servicios públicos"
      }
    ]
  },
  "sign": "hash_generado"
}
```

**Campos del detalle**:
- `account`: Código de cuenta contable
- `nature`: `D` (Débito) | `C` (Crédito)
- `value`: Valor numérico
- `costCenter`: Centro de costo (opcional)
- `description`: Descripción del movimiento

---

#### 2. Movimientos de Inventario
**Endpoint**: `POST /KansasWS/inventoryEntry`

Registra entradas de mercancía.

**Payload**:
```json
{
  "documentType": "EM",
  "warehouse": "BOD01",
  "date": {"day": 15, "month": 11, "year": 2024},
  "products": [
    {
      "code": "PROD001",
      "quantity": 100,
      "unitCost": 15000,
      "total": 1500000
    }
  ]
}
```

---

#### 3. Órdenes de Compra
**Endpoint**: `POST /KansasWS/purchaseOrder`

Crea órdenes de compra a proveedores.

**Información requerida**:
- Proveedor
- Fecha de orden
- Fecha esperada de entrega
- Productos y cantidades
- Condiciones de pago

---

#### 4. Órdenes de Venta
**Endpoint**: `POST /KansasWS/salesOrder`

Registra órdenes de venta a clientes.

**Información requerida**:
- Cliente
- Fecha de orden
- Productos y cantidades
- Precios y descuentos
- Condiciones de pago

---

#### 5. Productos Maestros
**Endpoint**: `POST /KansasWS/product`

Crea o actualiza registros maestros de productos.

**Campos principales**:
- Código (único)
- Descripción
- Familia/Subfamilia
- Unidad de medida
- Precios
- IVA aplicable
- Estado

---

#### 6. Remisiones
**Endpoint**: `POST /KansasWS/remission`

Crea documentos de envío/despacho.

**Información incluida**:
- Cliente o destinatario
- Productos y cantidades
- Bodega de origen
- Fecha de despacho
- Transportador (opcional)

---

#### 7. Proveedores y Clientes
**Endpoint**: `POST /KansasWS/supplier` | `POST /KansasWS/customer`

Crea o actualiza información de terceros.

**Campos requeridos**:
- Tipo de identificación
- Número de identificación
- Nombre o razón social
- Dirección
- Ciudad
- Teléfono
- Email
- Régimen tributario

---

## Modelos de Datos

### Estructura de Fechas

Todas las fechas en la API utilizan objetos con componentes separados:

```json
{
  "day": 15,
  "month": 11,
  "year": 2024
}
```

### Campos Enumerados

#### Naturaleza de Cuenta
- `D`: Débito
- `C`: Crédito

#### Tipos de Documento
- `CE`: Comprobante de Egreso
- `CI`: Comprobante de Ingreso
- `NC`: Nota Crédito
- `ND`: Nota Débito
- `FV`: Factura de Venta
- `FC`: Factura de Compra
- `EM`: Entrada de Mercancía
- `SM`: Salida de Mercancía

#### Regímenes Tributarios
- `SIMPLIFICADO`: Régimen simplificado
- `COMUN`: Régimen común
- `GRAN_CONTRIBUYENTE`: Gran contribuyente
- `NO_RESPONSABLE`: No responsable de IVA

### Estructuras de Transacciones

Las transacciones incluyen arrays anidados con:
- **Centros de costo**: Asignación de gastos/ingresos
- **Información tributaria**: IVA, retenciones, etc.
- **Detalles de productos**: En movimientos de inventario
- **Referencias cruzadas**: Vínculos entre documentos relacionados

**Ejemplo de transacción completa**:
```json
{
  "documentType": "FV",
  "number": "FV-001234",
  "date": {"day": 15, "month": 11, "year": 2024},
  "customer": {
    "id": "800123456",
    "name": "Cliente Ejemplo S.A.S."
  },
  "details": [
    {
      "product": "PROD001",
      "description": "Producto de ejemplo",
      "quantity": 10,
      "unitPrice": 50000,
      "discount": 0,
      "subtotal": 500000,
      "vat": 95000,
      "total": 595000,
      "costCenter": "CC001"
    }
  ],
  "taxes": {
    "subtotal": 500000,
    "vat": 95000,
    "retention": 0,
    "total": 595000
  }
}
```

---

## Manejo de Errores

### Códigos de Estado HTTP

La API utiliza códigos de estado HTTP estándar:

| Código | Significado | Descripción |
|--------|-------------|-------------|
| **200** | OK | Operación exitosa |
| **201** | Created | Recurso creado exitosamente |
| **400** | Bad Request | Error en la solicitud (datos inválidos) |
| **401** | Unauthorized | Error de autenticación (firma inválida) |
| **404** | Not Found | Recurso no encontrado |
| **500** | Internal Server Error | Error del servidor |

### Estructura de Respuestas de Error

```json
{
  "error": {
    "code": 1001,
    "message": "Firma de seguridad inválida",
    "details": "La firma proporcionada no coincide con la calculada"
  }
}
```

### Códigos de Error Comunes

| Código | Descripción |
|--------|-------------|
| **1001** | Firma de seguridad inválida |
| **1002** | Parámetro requerido faltante |
| **1003** | Formato de JSON inválido |
| **2001** | Cuenta contable no existe |
| **2002** | Producto no encontrado |
| **2003** | Centro de costo inválido |
| **3001** | Sumas débito/crédito no cuadran |
| **3002** | Documento duplicado |
| **4001** | Permisos insuficientes |

### Validaciones de Negocio

La API implementa validaciones estrictas:

1. **Comprobantes contables**: Las sumas de débitos deben igualar las sumas de créditos
2. **Inventario**: No se permiten cantidades negativas en existencias
3. **Documentos**: No se permite duplicación de números de documento
4. **Fechas**: No se permiten fechas futuras en la mayoría de operaciones
5. **Terceros**: Validación de NIT/RUT según formato legal

---

## Mejores Prácticas

### 1. Seguridad
- ✅ Siempre generar firmas con el algoritmo correcto
- ✅ Mantener las claves secretas seguras y rotarlas periódicamente
- ✅ Usar HTTPS para todas las comunicaciones
- ✅ No exponer información sensible en logs

### 2. Manejo de Datos
- ✅ Validar datos antes de enviar a la API
- ✅ Implementar reintentos con backoff exponencial
- ✅ Mantener registro de transacciones enviadas/pendientes
- ✅ Implementar idempotencia en operaciones críticas

### 3. Rendimiento
- ✅ Cachear catálogos que cambian poco (cuentas, centros de costo)
- ✅ Realizar inserciones en lotes cuando sea posible
- ✅ Implementar paginación en consultas grandes
- ✅ Monitorear tiempos de respuesta

### 4. Monitoreo
- 📊 Registrar todas las respuestas de error para análisis
- 📊 Implementar alertas para fallos recurrentes
- 📊 Mantener métricas de uso de la API
- 📊 Auditar todas las transacciones financieras

---

## Recursos Adicionales

- **Documentación de Arquitectura**: https://helisa.com/ayudas/api/Ayuda/descripcion_arquitectura/descripcion_arquitectura.html
- **Documentación de API**: https://helisa.com/api/

---

## Notas de Implementación

### Consideraciones Importantes

1. **Integridad de Datos**: El sistema implementa un seguimiento riguroso de registros enviados vs. no enviados para prevenir duplicación de transacciones.

2. **Sincronización Bidireccional**: Es esencial implementar correctamente ambos flujos (envío y recepción) para mantener sincronizados los sistemas.

3. **Firma de Seguridad**: La firma es obligatoria en TODAS las solicitudes. Un error en el cálculo de la firma resultará en rechazo de la petición.

4. **Validación Contable**: Los comprobantes contables deben cuadrar (débitos = créditos) o serán rechazados por el sistema.

5. **Manejo de Estados**: Es crítico marcar correctamente los registros como "enviados" solo después de recibir confirmación exitosa del servidor.

---

**Documento generado**: 2024-11-08
**Versión**: 1.0
**Fuentes**: Documentación oficial de Helisa
