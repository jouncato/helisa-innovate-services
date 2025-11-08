# Firma de Seguridad de Helisa API

## Tabla de Contenidos
- [Introducción](#introducción)
- [Propósito de la Firma de Seguridad](#propósito-de-la-firma-de-seguridad)
- [Algoritmo y Método](#algoritmo-y-método)
- [Parámetros Requeridos](#parámetros-requeridos)
- [Proceso de Implementación](#proceso-de-implementación)
- [Flujo de Validación](#flujo-de-validación)
- [Ejemplos de Código](#ejemplos-de-código)
- [Casos de Uso](#casos-de-uso)
- [Solución de Problemas](#solución-de-problemas)
- [Mejores Prácticas de Seguridad](#mejores-prácticas-de-seguridad)

---

## Introducción

La **firma de seguridad** es un componente crítico del sistema de autenticación de la API de Helisa. Este mecanismo garantiza que todas las comunicaciones entre aplicaciones externas y el WebService de Helisa sean seguras, autenticadas y no hayan sido alteradas durante la transmisión.

---

## Propósito de la Firma de Seguridad

La firma de seguridad cumple dos funciones esenciales:

### 1. 🔐 Autenticación
Garantiza que el cliente que hace la petición al WebService tiene la autorización adecuada para acceder a los recursos solicitados.

### 2. ✅ Integridad de Datos
Verifica que los datos transmitidos no han sido:
- Corrompidos durante la transmisión
- Alterados de forma maliciosa por terceros
- Modificados accidentalmente

> **Importante**: TODAS las peticiones a la API de Helisa deben incluir una firma de seguridad válida. Las peticiones sin firma o con firma inválida serán rechazadas.

---

## Algoritmo y Método

### Especificación Técnica

> **"El firmado de la información se hará por medio de HMAC utilizando la función de hash SHA-256."**

El sistema utiliza **HMAC-SHA256** (Hash-based Message Authentication Code con SHA-256) para la autenticación criptográfica.

### ¿Qué es HMAC-SHA256?

**HMAC** (Hash-based Message Authentication Code) es un algoritmo que combina:
- Una función de hash criptográfica (SHA-256 en este caso)
- Una clave secreta

Esta combinación produce una firma única que:
- ✅ Es prácticamente imposible de falsificar sin conocer la clave secreta
- ✅ Cambia completamente si se modifica cualquier bit de los datos
- ✅ Es determinista (los mismos datos + clave = misma firma)
- ✅ Es rápida de calcular y verificar

### Características de SHA-256

| Característica | Valor |
|----------------|-------|
| **Tamaño de hash** | 256 bits (32 bytes) |
| **Representación hexadecimal** | 64 caracteres |
| **Colisiones** | Computacionalmente inviables |
| **Velocidad** | Alta (optimizado en hardware moderno) |

---

## Parámetros Requeridos

Para generar una firma de seguridad válida, se necesitan tres componentes:

### 1. ID del Cliente (Client ID)
```
Tipo: String
Descripción: Identificador único asignado a cada cliente del WebService
Generalmente: ID de la empresa en el sistema Helisa
Ejemplo: "EMPRESA001"
```

**Características**:
- Proporcionado por Helisa al momento de la integración
- Único por cliente/empresa
- Debe mantenerse consistente en todas las peticiones

### 2. Clave Secreta (Secret Key)
```
Tipo: String (binario o hexadecimal)
Descripción: Clave criptográfica generada específicamente para el ID del cliente
Ejemplo: "a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6"
```

**Características**:
- Generada y proporcionada por Helisa
- Única para cada cliente
- **NUNCA debe compartirse o exponerse públicamente**
- Debe almacenarse de forma segura (variables de entorno, vault, etc.)
- Puede ser rotada periódicamente por seguridad

### 3. Datos JSON (JSON Payload)
```
Tipo: String JSON
Descripción: La información que se transmite en la petición
Ejemplo: {"cuenta": "110505", "tipo": "local"}
```

**Características**:
- Debe ser un JSON válido
- El formato y espacios en blanco deben ser consistentes
- Generalmente se envía sin formato (minificado) para evitar problemas

---

## Proceso de Implementación

### Diagrama de Flujo

```
┌─────────────────────────────────────────────────────────┐
│  CLIENTE                                                │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. Preparar datos JSON                                │
│     json = {"cuenta": "110505"}                         │
│                                                         │
│  2. Generar firma HMAC-SHA256                          │
│     signature = HMAC_SHA256(json, secretKey)           │
│                                                         │
│  3. Convertir a hexadecimal                            │
│     sign = toHex(signature)                            │
│                                                         │
│  4. Preparar petición                                   │
│     request = {                                         │
│       id: clientId,                                     │
│       json: json,                                       │
│       sign: sign                                        │
│     }                                                   │
│                                                         │
│  5. Enviar al WebService                               │
│     POST /KansasWS/endpoint                            │
│                                                         │
└────────────────┬────────────────────────────────────────┘
                 │
                 │ HTTP Request
                 ▼
┌─────────────────────────────────────────────────────────┐
│  SERVIDOR HELISA                                        │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. Recibir petición                                    │
│     Extraer: id, json, sign                            │
│                                                         │
│  2. Recuperar clave secreta                            │
│     secretKey = getSecretKey(id)                       │
│                                                         │
│  3. Generar firma del servidor                         │
│     serverSignature = HMAC_SHA256(json, secretKey)     │
│     serverSign = toHex(serverSignature)                │
│                                                         │
│  4. Comparar firmas                                     │
│     if (sign === serverSign) {                         │
│       ✅ Firma válida → Procesar petición              │
│     } else {                                            │
│       ❌ Firma inválida → Rechazar                     │
│     }                                                   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Pasos Detallados

#### Paso 1: Preparar los Datos JSON
```javascript
const data = {
  cuenta: "110505",
  tipo: "local"
};

// Convertir a string JSON (generalmente minificado)
const jsonString = JSON.stringify(data);
// Resultado: '{"cuenta":"110505","tipo":"local"}'
```

#### Paso 2: Generar el HMAC-SHA256
```javascript
const crypto = require('crypto');

const secretKey = 'tu_clave_secreta_proporcionada_por_helisa';
const signature = crypto
  .createHmac('sha256', secretKey)
  .update(jsonString)
  .digest();

// signature es un Buffer con datos binarios
```

#### Paso 3: Convertir a Hexadecimal
```javascript
// Convertir de binario a hexadecimal (64 caracteres)
const sign = signature.toString('hex');

// Ejemplo de resultado:
// "a7f3c9d2e1b4f8a6c3d5e7f9b2a4c6d8e1f3a5b7c9d2e4f6a8b1c3d5e7f9a2b4"
```

#### Paso 4: Preparar y Enviar la Petición
```javascript
const requestData = {
  id: 'EMPRESA001',      // Tu ID de cliente
  json: jsonString,       // Los datos en formato JSON string
  sign: sign              // La firma hexadecimal
};

// Enviar como form-data o application/x-www-form-urlencoded
const response = await fetch('https://helisa.com/api/KansasWS/endpoint', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded'
  },
  body: new URLSearchParams(requestData)
});
```

---

## Flujo de Validación

### En el Servidor Helisa

Cuando el servidor recibe una petición, sigue este proceso:

```javascript
// Pseudocódigo del proceso de validación en el servidor

function validateRequest(request) {
  // 1. Extraer parámetros
  const { id, json, sign } = request;

  // 2. Validar que todos los parámetros existan
  if (!id || !json || !sign) {
    return {
      error: true,
      code: 1002,
      message: "Parámetros requeridos faltantes"
    };
  }

  // 3. Obtener la clave secreta del cliente
  const secretKey = database.getSecretKey(id);

  if (!secretKey) {
    return {
      error: true,
      code: 4001,
      message: "Cliente no autorizado"
    };
  }

  // 4. Generar la firma del servidor
  const serverSignature = HMAC_SHA256(json, secretKey);
  const serverSign = toHex(serverSignature);

  // 5. Comparar firmas (comparación constante en tiempo)
  if (!constantTimeCompare(sign, serverSign)) {
    return {
      error: true,
      code: 1001,
      message: "Firma de seguridad inválida"
    };
  }

  // 6. Firma válida - procesar la petición
  return processRequest(json);
}
```

### Diagrama de Decisión

```
                    Petición recibida
                           │
                           ▼
                ┌──────────────────────┐
                │ ¿Existen id, json,   │
                │ sign?                │
                └──────────┬───────────┘
                           │
                    ┌──────┴──────┐
                    │             │
                   NO            SI
                    │             │
                    ▼             ▼
             ┌──────────┐  ┌──────────────┐
             │ Error    │  │ Obtener      │
             │ 1002     │  │ secretKey    │
             └──────────┘  └──────┬───────┘
                                  │
                           ┌──────┴──────┐
                           │             │
                       NULL           EXISTS
                           │             │
                           ▼             ▼
                    ┌──────────┐  ┌──────────────┐
                    │ Error    │  │ Generar      │
                    │ 4001     │  │ firma server │
                    └──────────┘  └──────┬───────┘
                                         │
                                         ▼
                                  ┌──────────────┐
                                  │ Comparar     │
                                  │ firmas       │
                                  └──────┬───────┘
                                         │
                                  ┌──────┴──────┐
                                  │             │
                              NO MATCH      MATCH
                                  │             │
                                  ▼             ▼
                           ┌──────────┐  ┌──────────────┐
                           │ Error    │  │ ✅ Procesar  │
                           │ 1001     │  │ petición     │
                           └──────────┘  └──────────────┘
```

---

## Ejemplos de Código

### JavaScript / Node.js

```javascript
const crypto = require('crypto');

/**
 * Genera una firma de seguridad HMAC-SHA256
 * @param {Object} data - Datos a firmar
 * @param {string} secretKey - Clave secreta
 * @param {string} clientId - ID del cliente
 * @returns {Object} Objeto con los parámetros para la petición
 */
function generateSignature(data, secretKey, clientId) {
  // Convertir datos a JSON string
  const jsonString = JSON.stringify(data);

  // Generar HMAC-SHA256
  const signature = crypto
    .createHmac('sha256', secretKey)
    .update(jsonString)
    .digest('hex');

  return {
    id: clientId,
    json: jsonString,
    sign: signature
  };
}

// Ejemplo de uso
const data = {
  documentType: "CE",
  date: { day: 15, month: 11, year: 2024 },
  description: "Comprobante de prueba"
};

const secretKey = process.env.HELISA_SECRET_KEY;
const clientId = process.env.HELISA_CLIENT_ID;

const requestParams = generateSignature(data, secretKey, clientId);

console.log('Parámetros de la petición:', requestParams);
/*
{
  id: 'EMPRESA001',
  json: '{"documentType":"CE","date":{"day":15,"month":11,"year":2024},"description":"Comprobante de prueba"}',
  sign: 'a7f3c9d2e1b4f8a6c3d5e7f9b2a4c6d8e1f3a5b7c9d2e4f6a8b1c3d5e7f9a2b4'
}
*/
```

### Python

```python
import hmac
import hashlib
import json
import os

def generate_signature(data, secret_key, client_id):
    """
    Genera una firma de seguridad HMAC-SHA256

    Args:
        data (dict): Datos a firmar
        secret_key (str): Clave secreta
        client_id (str): ID del cliente

    Returns:
        dict: Diccionario con los parámetros para la petición
    """
    # Convertir datos a JSON string
    json_string = json.dumps(data, separators=(',', ':'))

    # Generar HMAC-SHA256
    signature = hmac.new(
        secret_key.encode('utf-8'),
        json_string.encode('utf-8'),
        hashlib.sha256
    ).hexdigest()

    return {
        'id': client_id,
        'json': json_string,
        'sign': signature
    }

# Ejemplo de uso
if __name__ == '__main__':
    data = {
        'documentType': 'CE',
        'date': {'day': 15, 'month': 11, 'year': 2024},
        'description': 'Comprobante de prueba'
    }

    secret_key = os.getenv('HELISA_SECRET_KEY')
    client_id = os.getenv('HELISA_CLIENT_ID')

    request_params = generate_signature(data, secret_key, client_id)

    print('Parámetros de la petición:', request_params)
```

### PHP

```php
<?php

/**
 * Genera una firma de seguridad HMAC-SHA256
 *
 * @param array $data Datos a firmar
 * @param string $secretKey Clave secreta
 * @param string $clientId ID del cliente
 * @return array Arreglo con los parámetros para la petición
 */
function generateSignature($data, $secretKey, $clientId) {
    // Convertir datos a JSON string
    $jsonString = json_encode($data, JSON_UNESCAPED_SLASHES | JSON_UNESCAPED_UNICODE);

    // Generar HMAC-SHA256
    $signature = hash_hmac('sha256', $jsonString, $secretKey);

    return [
        'id' => $clientId,
        'json' => $jsonString,
        'sign' => $signature
    ];
}

// Ejemplo de uso
$data = [
    'documentType' => 'CE',
    'date' => ['day' => 15, 'month' => 11, 'year' => 2024],
    'description' => 'Comprobante de prueba'
];

$secretKey = getenv('HELISA_SECRET_KEY');
$clientId = getenv('HELISA_CLIENT_ID');

$requestParams = generateSignature($data, $secretKey, $clientId);

print_r($requestParams);
?>
```

### Java

```java
import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;
import java.nio.charset.StandardCharsets;
import com.google.gson.Gson;
import java.util.HashMap;
import java.util.Map;

public class HelisaSignature {

    /**
     * Genera una firma de seguridad HMAC-SHA256
     *
     * @param data Datos a firmar
     * @param secretKey Clave secreta
     * @param clientId ID del cliente
     * @return Map con los parámetros para la petición
     */
    public static Map<String, String> generateSignature(
            Object data,
            String secretKey,
            String clientId) throws Exception {

        // Convertir datos a JSON string
        Gson gson = new Gson();
        String jsonString = gson.toJson(data);

        // Generar HMAC-SHA256
        Mac sha256Hmac = Mac.getInstance("HmacSHA256");
        SecretKeySpec secretKeySpec = new SecretKeySpec(
            secretKey.getBytes(StandardCharsets.UTF_8),
            "HmacSHA256"
        );
        sha256Hmac.init(secretKeySpec);

        byte[] signatureBytes = sha256Hmac.doFinal(
            jsonString.getBytes(StandardCharsets.UTF_8)
        );

        // Convertir a hexadecimal
        String signature = bytesToHex(signatureBytes);

        // Crear mapa de respuesta
        Map<String, String> requestParams = new HashMap<>();
        requestParams.put("id", clientId);
        requestParams.put("json", jsonString);
        requestParams.put("sign", signature);

        return requestParams;
    }

    /**
     * Convierte array de bytes a string hexadecimal
     */
    private static String bytesToHex(byte[] bytes) {
        StringBuilder result = new StringBuilder();
        for (byte b : bytes) {
            result.append(String.format("%02x", b));
        }
        return result.toString();
    }

    // Ejemplo de uso
    public static void main(String[] args) throws Exception {
        Map<String, Object> data = new HashMap<>();
        data.put("documentType", "CE");

        Map<String, Integer> date = new HashMap<>();
        date.put("day", 15);
        date.put("month", 11);
        date.put("year", 2024);
        data.put("date", date);

        data.put("description", "Comprobante de prueba");

        String secretKey = System.getenv("HELISA_SECRET_KEY");
        String clientId = System.getenv("HELISA_CLIENT_ID");

        Map<String, String> requestParams = generateSignature(
            data,
            secretKey,
            clientId
        );

        System.out.println("Parámetros de la petición:");
        requestParams.forEach((key, value) ->
            System.out.println(key + ": " + value)
        );
    }
}
```

### C# / .NET

```csharp
using System;
using System.Security.Cryptography;
using System.Text;
using System.Text.Json;
using System.Collections.Generic;

public class HelisaSignature
{
    /// <summary>
    /// Genera una firma de seguridad HMAC-SHA256
    /// </summary>
    /// <param name="data">Datos a firmar</param>
    /// <param name="secretKey">Clave secreta</param>
    /// <param name="clientId">ID del cliente</param>
    /// <returns>Diccionario con los parámetros para la petición</returns>
    public static Dictionary<string, string> GenerateSignature(
        object data,
        string secretKey,
        string clientId)
    {
        // Convertir datos a JSON string
        var jsonString = JsonSerializer.Serialize(data, new JsonSerializerOptions
        {
            PropertyNamingPolicy = JsonNamingPolicy.CamelCase
        });

        // Generar HMAC-SHA256
        using (var hmac = new HMACSHA256(Encoding.UTF8.GetBytes(secretKey)))
        {
            var signatureBytes = hmac.ComputeHash(Encoding.UTF8.GetBytes(jsonString));
            var signature = BitConverter.ToString(signatureBytes)
                .Replace("-", "")
                .ToLower();

            return new Dictionary<string, string>
            {
                { "id", clientId },
                { "json", jsonString },
                { "sign", signature }
            };
        }
    }

    // Ejemplo de uso
    public static void Main(string[] args)
    {
        var data = new
        {
            DocumentType = "CE",
            Date = new { Day = 15, Month = 11, Year = 2024 },
            Description = "Comprobante de prueba"
        };

        var secretKey = Environment.GetEnvironmentVariable("HELISA_SECRET_KEY");
        var clientId = Environment.GetEnvironmentVariable("HELISA_CLIENT_ID");

        var requestParams = GenerateSignature(data, secretKey, clientId);

        Console.WriteLine("Parámetros de la petición:");
        foreach (var param in requestParams)
        {
            Console.WriteLine($"{param.Key}: {param.Value}");
        }
    }
}
```

### Go

```go
package main

import (
    "crypto/hmac"
    "crypto/sha256"
    "encoding/hex"
    "encoding/json"
    "fmt"
    "os"
)

// RequestParams representa los parámetros de la petición
type RequestParams struct {
    ID   string `json:"id"`
    JSON string `json:"json"`
    Sign string `json:"sign"`
}

// GenerateSignature genera una firma de seguridad HMAC-SHA256
func GenerateSignature(data interface{}, secretKey, clientID string) (*RequestParams, error) {
    // Convertir datos a JSON string
    jsonBytes, err := json.Marshal(data)
    if err != nil {
        return nil, err
    }
    jsonString := string(jsonBytes)

    // Generar HMAC-SHA256
    h := hmac.New(sha256.New, []byte(secretKey))
    h.Write([]byte(jsonString))
    signature := hex.EncodeToString(h.Sum(nil))

    return &RequestParams{
        ID:   clientID,
        JSON: jsonString,
        Sign: signature,
    }, nil
}

// Ejemplo de uso
func main() {
    data := map[string]interface{}{
        "documentType": "CE",
        "date": map[string]int{
            "day":   15,
            "month": 11,
            "year":  2024,
        },
        "description": "Comprobante de prueba",
    }

    secretKey := os.Getenv("HELISA_SECRET_KEY")
    clientID := os.Getenv("HELISA_CLIENT_ID")

    requestParams, err := GenerateSignature(data, secretKey, clientID)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }

    fmt.Printf("Parámetros de la petición:\n")
    fmt.Printf("ID: %s\n", requestParams.ID)
    fmt.Printf("JSON: %s\n", requestParams.JSON)
    fmt.Printf("Sign: %s\n", requestParams.Sign)
}
```

---

## Casos de Uso

### Caso 1: Consulta de Cuentas Contables

```javascript
const crypto = require('crypto');
const axios = require('axios');

async function getCuentasContables() {
  const data = {
    tipo: "local"
  };

  const jsonString = JSON.stringify(data);
  const secretKey = process.env.HELISA_SECRET_KEY;
  const clientId = process.env.HELISA_CLIENT_ID;

  const sign = crypto
    .createHmac('sha256', secretKey)
    .update(jsonString)
    .digest('hex');

  try {
    const response = await axios.post(
      'https://helisa.com/api/KansasWS/cuentas',
      new URLSearchParams({
        id: clientId,
        json: jsonString,
        sign: sign
      }),
      {
        headers: {
          'Content-Type': 'application/x-www-form-urlencoded'
        }
      }
    );

    return response.data;
  } catch (error) {
    console.error('Error al consultar cuentas:', error.response?.data);
    throw error;
  }
}
```

### Caso 2: Inserción de Comprobante Contable

```javascript
async function insertarComprobante() {
  const data = {
    documentType: "CE",
    date: { day: 15, month: 11, year: 2024 },
    description: "Pago de servicios públicos",
    details: [
      {
        account: "110505",
        nature: "C",
        value: 500000,
        costCenter: "CC001",
        description: "Pago efectivo"
      },
      {
        account: "510506",
        nature: "D",
        value: 500000,
        costCenter: "CC001",
        description: "Gasto servicios"
      }
    ]
  };

  const jsonString = JSON.stringify(data);
  const secretKey = process.env.HELISA_SECRET_KEY;
  const clientId = process.env.HELISA_CLIENT_ID;

  const sign = crypto
    .createHmac('sha256', secretKey)
    .update(jsonString)
    .digest('hex');

  try {
    const response = await axios.post(
      'https://helisa.com/api/KansasWS/comprobante',
      new URLSearchParams({
        id: clientId,
        json: jsonString,
        sign: sign
      }),
      {
        headers: {
          'Content-Type': 'application/x-www-form-urlencoded'
        }
      }
    );

    console.log('Comprobante insertado exitosamente:', response.data);
    return response.data;
  } catch (error) {
    console.error('Error al insertar comprobante:', error.response?.data);
    throw error;
  }
}
```

### Caso 3: Wrapper Reutilizable

```javascript
class HelisaClient {
  constructor(clientId, secretKey) {
    this.clientId = clientId;
    this.secretKey = secretKey;
    this.baseUrl = 'https://helisa.com/api/KansasWS';
  }

  /**
   * Genera la firma de seguridad para los datos
   */
  generateSignature(data) {
    const jsonString = JSON.stringify(data);
    const sign = crypto
      .createHmac('sha256', this.secretKey)
      .update(jsonString)
      .digest('hex');

    return {
      id: this.clientId,
      json: jsonString,
      sign: sign
    };
  }

  /**
   * Realiza una petición a la API de Helisa
   */
  async request(endpoint, data = {}) {
    const params = this.generateSignature(data);

    try {
      const response = await axios.post(
        `${this.baseUrl}/${endpoint}`,
        new URLSearchParams(params),
        {
          headers: {
            'Content-Type': 'application/x-www-form-urlencoded'
          }
        }
      );

      return response.data;
    } catch (error) {
      if (error.response?.data?.error?.code === 1001) {
        throw new Error('Firma de seguridad inválida. Verifica tu clave secreta.');
      }
      throw error;
    }
  }

  /**
   * Métodos específicos de la API
   */
  async getCuentas(tipo = 'local') {
    return this.request('cuentas', { tipo });
  }

  async getProductos() {
    return this.request('productos');
  }

  async insertarComprobante(comprobante) {
    return this.request('comprobante', comprobante);
  }
}

// Uso
const client = new HelisaClient(
  process.env.HELISA_CLIENT_ID,
  process.env.HELISA_SECRET_KEY
);

// Consultar cuentas
const cuentas = await client.getCuentas('local');

// Insertar comprobante
const resultado = await client.insertarComprobante({
  documentType: "CE",
  date: { day: 15, month: 11, year: 2024 },
  description: "Pago de servicios",
  details: [...]
});
```

---

## Solución de Problemas

### Error: Firma de seguridad inválida (Código 1001)

**Causas comunes**:

1. **Clave secreta incorrecta**
   ```javascript
   // ❌ Incorrecto
   const secretKey = 'mi_clave_inventada';

   // ✅ Correcto
   const secretKey = process.env.HELISA_SECRET_KEY; // Proporcionada por Helisa
   ```

2. **Formato JSON inconsistente**
   ```javascript
   // ❌ Incorrecto - espacios extra
   const json = JSON.stringify(data, null, 2);

   // ✅ Correcto - minificado
   const json = JSON.stringify(data);
   ```

3. **Encoding incorrecto**
   ```javascript
   // ✅ Asegúrate de usar UTF-8
   const signature = crypto
     .createHmac('sha256', secretKey)
     .update(jsonString, 'utf8')  // Especificar encoding
     .digest('hex');
   ```

4. **Modificación del JSON después de firmar**
   ```javascript
   // ❌ Incorrecto
   const sign = generateSignature(data);
   data.newField = 'value'; // ¡No modificar después de firmar!
   sendRequest(data, sign);

   // ✅ Correcto
   const sign = generateSignature(data);
   sendRequest(data, sign); // Usar los mismos datos
   ```

### Error: Parámetros requeridos faltantes (Código 1002)

**Solución**: Verificar que todos los parámetros estén presentes

```javascript
// Verificación antes de enviar
function validateParams(params) {
  if (!params.id) {
    throw new Error('Falta el parámetro "id"');
  }
  if (!params.json) {
    throw new Error('Falta el parámetro "json"');
  }
  if (!params.sign) {
    throw new Error('Falta el parámetro "sign"');
  }
  return true;
}
```

### Error: Cliente no autorizado (Código 4001)

**Causas**:
- ID de cliente incorrecto
- Cliente no registrado en el sistema
- Permisos revocados

**Solución**: Verificar con Helisa que el ID de cliente sea correcto y esté activo.

### Debugging de Firmas

```javascript
function debugSignature(data, secretKey, clientId) {
  const jsonString = JSON.stringify(data);

  console.log('=== DEBUG FIRMA DE SEGURIDAD ===');
  console.log('Client ID:', clientId);
  console.log('JSON String:', jsonString);
  console.log('JSON Length:', jsonString.length);
  console.log('Secret Key Length:', secretKey.length);

  const signature = crypto
    .createHmac('sha256', secretKey)
    .update(jsonString)
    .digest('hex');

  console.log('Signature:', signature);
  console.log('Signature Length:', signature.length); // Debe ser 64
  console.log('================================');

  return {
    id: clientId,
    json: jsonString,
    sign: signature
  };
}
```

### Comparación de Firmas

Para debugging, puedes comparar la firma que generas con la que debería generarse:

```javascript
function compareSignatures(expected, actual) {
  console.log('Firma esperada:', expected);
  console.log('Firma generada:', actual);
  console.log('¿Coinciden?:', expected === actual);

  if (expected !== actual) {
    // Comparar carácter por carácter
    for (let i = 0; i < Math.max(expected.length, actual.length); i++) {
      if (expected[i] !== actual[i]) {
        console.log(`Diferencia en posición ${i}:`);
        console.log(`  Esperado: "${expected[i]}" (${expected.charCodeAt(i)})`);
        console.log(`  Actual: "${actual[i]}" (${actual.charCodeAt(i)})`);
        break;
      }
    }
  }
}
```

---

## Mejores Prácticas de Seguridad

### 1. 🔐 Almacenamiento de Claves

**❌ NUNCA hacer**:
```javascript
// NO almacenar claves en el código
const secretKey = 'a1b2c3d4e5f6...';

// NO versionar claves en Git
// NO compartir claves en mensajes o emails
// NO almacenar claves en logs
```

**✅ Hacer**:
```javascript
// Usar variables de entorno
const secretKey = process.env.HELISA_SECRET_KEY;

// Usar servicios de gestión de secretos
// AWS Secrets Manager, Azure Key Vault, HashiCorp Vault, etc.

// En .env (nunca versionar este archivo)
HELISA_SECRET_KEY=tu_clave_secreta_aqui
HELISA_CLIENT_ID=tu_client_id_aqui
```

**Archivo .env.example** (sí versionar):
```bash
# Configuración de Helisa API
HELISA_SECRET_KEY=tu_clave_aqui
HELISA_CLIENT_ID=tu_id_aqui
HELISA_API_URL=https://helisa.com/api/KansasWS
```

**Archivo .gitignore**:
```
.env
.env.local
.env.*.local
secrets/
*.key
```

### 2. 🔄 Rotación de Claves

```javascript
class KeyRotationManager {
  constructor() {
    this.currentKey = process.env.HELISA_SECRET_KEY;
    this.previousKey = process.env.HELISA_SECRET_KEY_OLD;
  }

  /**
   * Genera firma con la clave actual
   */
  generateSignature(data) {
    const jsonString = JSON.stringify(data);
    return crypto
      .createHmac('sha256', this.currentKey)
      .update(jsonString)
      .digest('hex');
  }

  /**
   * Verifica firma con clave actual y anterior (durante período de transición)
   */
  verifySignature(data, signature) {
    const jsonString = JSON.stringify(data);

    // Probar con clave actual
    const currentSignature = crypto
      .createHmac('sha256', this.currentKey)
      .update(jsonString)
      .digest('hex');

    if (currentSignature === signature) {
      return { valid: true, keyUsed: 'current' };
    }

    // Si falla, probar con clave anterior (durante migración)
    if (this.previousKey) {
      const previousSignature = crypto
        .createHmac('sha256', this.previousKey)
        .update(jsonString)
        .digest('hex');

      if (previousSignature === signature) {
        return { valid: true, keyUsed: 'previous', warning: 'Usar clave nueva' };
      }
    }

    return { valid: false };
  }
}
```

### 3. 🛡️ Protección contra Timing Attacks

```javascript
const crypto = require('crypto');

/**
 * Comparación de strings en tiempo constante
 * Previene timing attacks
 */
function constantTimeCompare(a, b) {
  // Usar la función nativa de Node.js si está disponible
  if (crypto.timingSafeEqual) {
    try {
      return crypto.timingSafeEqual(
        Buffer.from(a, 'utf8'),
        Buffer.from(b, 'utf8')
      );
    } catch {
      return false;
    }
  }

  // Fallback manual
  if (a.length !== b.length) {
    return false;
  }

  let result = 0;
  for (let i = 0; i < a.length; i++) {
    result |= a.charCodeAt(i) ^ b.charCodeAt(i);
  }

  return result === 0;
}
```

### 4. 📝 Logging Seguro

```javascript
const winston = require('winston');

// Crear logger que oculta información sensible
const logger = winston.createLogger({
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json(),
    winston.format((info) => {
      // Ocultar claves secretas
      if (info.secretKey) {
        info.secretKey = '***REDACTED***';
      }
      // Ocultar firmas completas (mostrar solo inicio)
      if (info.sign && info.sign.length > 10) {
        info.sign = info.sign.substring(0, 8) + '...';
      }
      return info;
    })()
  ),
  transports: [
    new winston.transports.File({ filename: 'helisa-api.log' })
  ]
});

// Uso
logger.info('Generando firma', {
  clientId: 'EMPRESA001',
  dataLength: jsonString.length,
  sign: signature // Será truncado automáticamente
});
```

### 5. ⚠️ Validación de Entrada

```javascript
function validateAndSanitizeData(data) {
  // Validar que data sea un objeto
  if (typeof data !== 'object' || data === null) {
    throw new Error('Los datos deben ser un objeto válido');
  }

  // Validar tamaño del JSON
  const jsonString = JSON.stringify(data);
  const maxSize = 1024 * 1024; // 1MB

  if (jsonString.length > maxSize) {
    throw new Error(`Datos demasiado grandes (máx ${maxSize} bytes)`);
  }

  // Eliminar caracteres potencialmente peligrosos
  // (dependiendo de tus requisitos)

  return data;
}
```

### 6. 🔒 HTTPS Obligatorio

```javascript
const axios = require('axios');

// Configurar cliente con validación estricta de SSL
const httpsAgent = new https.Agent({
  rejectUnauthorized: true, // Rechazar certificados inválidos
  minVersion: 'TLSv1.2'      // Usar TLS 1.2 o superior
});

const helisaClient = axios.create({
  baseURL: 'https://helisa.com/api/KansasWS',
  httpsAgent: httpsAgent,
  timeout: 30000
});

// Verificar que la URL sea HTTPS
function ensureHttps(url) {
  if (!url.startsWith('https://')) {
    throw new Error('Solo se permiten conexiones HTTPS');
  }
  return url;
}
```

### 7. 🎯 Principio de Privilegios Mínimos

```javascript
// Separar credenciales por entorno
const config = {
  development: {
    clientId: process.env.HELISA_DEV_CLIENT_ID,
    secretKey: process.env.HELISA_DEV_SECRET_KEY,
    baseUrl: 'https://helisa-dev.com/api/KansasWS'
  },
  production: {
    clientId: process.env.HELISA_PROD_CLIENT_ID,
    secretKey: process.env.HELISA_PROD_SECRET_KEY,
    baseUrl: 'https://helisa.com/api/KansasWS'
  }
};

const env = process.env.NODE_ENV || 'development';
const helisaConfig = config[env];
```

### 8. 📊 Auditoría y Monitoreo

```javascript
class AuditLogger {
  constructor() {
    this.logger = winston.createLogger({
      transports: [
        new winston.transports.File({
          filename: 'helisa-audit.log',
          format: winston.format.json()
        })
      ]
    });
  }

  logRequest(endpoint, data, signature) {
    this.logger.info('API Request', {
      timestamp: new Date().toISOString(),
      endpoint: endpoint,
      dataHash: crypto.createHash('sha256').update(JSON.stringify(data)).digest('hex'),
      signaturePrefix: signature.substring(0, 8),
      clientId: process.env.HELISA_CLIENT_ID
    });
  }

  logResponse(endpoint, success, error = null) {
    this.logger.info('API Response', {
      timestamp: new Date().toISOString(),
      endpoint: endpoint,
      success: success,
      error: error ? {
        code: error.code,
        message: error.message
      } : null
    });
  }

  logSecurityEvent(event, details) {
    this.logger.warn('Security Event', {
      timestamp: new Date().toISOString(),
      event: event,
      details: details
    });
  }
}

// Uso
const auditLogger = new AuditLogger();

// Registrar evento de seguridad
if (signatureInvalid) {
  auditLogger.logSecurityEvent('INVALID_SIGNATURE', {
    clientId: clientId,
    endpoint: endpoint
  });
}
```

---

## Checklist de Implementación

Antes de desplegar a producción, verifica:

- [ ] ✅ Claves secretas almacenadas en variables de entorno
- [ ] ✅ Claves secretas NO versionadas en Git
- [ ] ✅ Uso de HTTPS para todas las comunicaciones
- [ ] ✅ Validación de entrada de datos
- [ ] ✅ Manejo adecuado de errores de firma
- [ ] ✅ Logging que NO expone información sensible
- [ ] ✅ Implementación de reintentos con backoff
- [ ] ✅ Monitoreo de errores de autenticación
- [ ] ✅ Pruebas de integración con la API
- [ ] ✅ Documentación para el equipo
- [ ] ✅ Plan de rotación de claves
- [ ] ✅ Auditoría de transacciones críticas

---

## Referencias

- **Documentación oficial**: https://helisa.com/ayudas/api/Ayuda/firma-de-seguridad/firma-de-seguridad.html
- **RFC 2104**: HMAC: Keyed-Hashing for Message Authentication
- **FIPS 180-4**: Secure Hash Standard (SHA-256)
- **OWASP**: Key Management Cheat Sheet

---

## Conclusión

La firma de seguridad HMAC-SHA256 es un componente crítico para garantizar la autenticación e integridad de las comunicaciones con la API de Helisa.

**Puntos clave a recordar**:

1. 🔐 **Nunca** expongas o compartas tu clave secreta
2. ✅ **Siempre** genera la firma antes de cada petición
3. 🔄 **Implementa** rotación periódica de claves
4. 📝 **Registra** eventos de seguridad sin exponer datos sensibles
5. 🛡️ **Valida** todas las respuestas del servidor

---

**Documento generado**: 2024-11-08
**Versión**: 1.0
**Fuente**: Documentación oficial de Helisa
