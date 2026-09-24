# NetSegment API

> Microservicio para segmentación, asignación y agregación de direccionamiento IPv4 mediante FLSM, VLSM y Supernetting (CIDR).

[![Estado](https://img.shields.io/badge/estado-en%20desarrollo-yellow)]()
[![TRL](https://img.shields.io/badge/TRL-3%20(prueba%20de%20concepto)-blue)]()
[![Python](https://img.shields.io/badge/python-3.11%2B-blue)]()
[![Framework](https://img.shields.io/badge/framework-FastAPI-009688)]()

## ⚠️ Estado del proyecto

Este proyecto se encuentra actualmente en **TRL 3 (prueba de concepto a nivel analítico y de diseño)**. Esto significa que:

- ✅ La arquitectura, los esquemas de datos y la lógica algorítmica están **completamente diseñados y validados analíticamente**.
- ✅ El contrato de API (endpoints, formatos de request/response, códigos de error) está **definido y documentado**.

Este README describe ese diseño: la arquitectura del microservicio y el contrato de API tal como fueron validados analíticamente.

## 📋 Descripción

En la gestión de redes, calcular subredes manualmente (hojas de cálculo, calculadoras web) genera errores frecuentes: solapamiento de direcciones (IP overlap), desperdicio de bloques IPv4 y sobrecarga de tablas de enrutamiento por falta de sumarización.

**NetSegment API** resuelve esto exponiendo un microservicio REST que automatiza:

- **FLSM** (Fixed Length Subnet Mask) — división de una red en subredes de igual tamaño.
- **VLSM** (Variable Length Subnet Mask) — asignación óptima de bloques según demanda real de hosts, usando un algoritmo voraz (greedy).
- **Supernetting / CIDR** — agregación de rutas contiguas en una sola entrada de enrutamiento.

A diferencia de las calculadoras web tradicionales, está pensado para ser **consumido por otro software**: pipelines de CI/CD, scripts de Ansible/Terraform, o cualquier sistema de aprovisionamiento de infraestructura (IaC).

## 🏗️ Arquitectura

```
Cliente HTTP (cURL / Postman / Script IaC / Frontend)
        │  POST (JSON)
        ▼
Servidor ASGI (Uvicorn, TCP:8000)
        │
        ▼
Capa de enrutamiento (FastAPI Router)
        │
        ▼
Validación de esquemas (Pydantic v2) ──✗──> 422 Unprocessable Entity
        │ ✓
        ▼
Selector de operación (/flsm | /vlsm | /aggregate)
        │
        ▼
Motor lógico de red (módulo ipaddress, álgebra de Boole)
        │
        ▼
Validación de límites y capacidad ──✗──> 400 Bad Request
        │ ✓
        ▼
Respuesta 200 OK (JSON con detalle de red)
```

## 🛠️ Stack tecnológico

| Componente | Tecnología |
|---|---|
| Lenguaje | Python 3.11+ |
| Framework web | FastAPI (ASGI) |
| Validación de datos | Pydantic v2 |
| Cálculo de direcciones | Módulo nativo `ipaddress` |
| Servidor | Uvicorn (uvloop + httptools) |
| Aislamiento de dependencias | `venv` |

## 💻 Requisitos

### Mínimos
- 1 vCPU (x86_64 o ARM64) a 1.5 GHz
- 512 MB de RAM
- 500 MB libres en disco
- Linux (Ubuntu Server 22.04 LTS / Debian 12 o cualquier POSIX)
- Puerto TCP 8000 disponible

### Recomendados
- 2 vCPUs a 2.0 GHz o superior
- 2 GB de RAM DDR4+
- 5 GB en SSD
- Ubuntu Server 24.04 LTS (contenedores)
- Proxy inverso (Nginx / Traefik) sobre puerto 80/443

## 📡 Contrato de API (diseño validado)

Todos los endpoints operan bajo el prefijo `/api/v1`, con método `POST` y JSON en UTF-8.

### 1. FLSM — Subnetting de longitud fija

`POST /api/v1/subnetting/flsm`

**Request:**
```json
{
  "network": "192.168.10.0/24",
  "subnets_needed": 4
}
```

**Response (200 OK):**
```json
{
  "status": "success",
  "data": {
    "base_network": "192.168.10.0/24",
    "cidr_prefix": 26,
    "usable_hosts_per_subnet": 62,
    "subnets": [
      {
        "subnet_id": 1,
        "network_address": "192.168.10.0",
        "cidr": "/26",
        "first_usable_ip": "192.168.10.1",
        "last_usable_ip": "192.168.10.62",
        "broadcast_address": "192.168.10.63"
      }
    ]
  }
}
```

### 2. VLSM — Subnetting de longitud variable

`POST /api/v1/subnetting/vlsm`

**Request:**
```json
{
  "base_network": "172.16.0.0/22",
  "departments": [
    { "name": "Ventas", "hosts_needed": 120 },
    { "name": "Ingenieria", "hosts_needed": 500 },
    { "name": "Servidores", "hosts_needed": 50 },
    { "name": "Enlace-WAN", "hosts_needed": 2 }
  ]
}
```

**Response (200 OK):**
```json
{
  "status": "success",
  "data": {
    "base_network": "172.16.0.0/22",
    "total_space_available": 1024,
    "total_space_allocated": 772,
    "allocations": [
      {
        "department": "Ingenieria",
        "requested_hosts": 500,
        "allocated_hosts": 510,
        "network_address": "172.16.0.0",
        "cidr": "/23"
      }
    ]
  }
}
```

> El algoritmo reordena automáticamente las demandas de mayor a menor antes de asignar los bloques, para evitar fragmentación.

### 3. Supernetting — Agregación de rutas CIDR

`POST /api/v1/supernetting/aggregate`

**Request:**
```json
{
  "networks": [
    "192.168.0.0/24",
    "192.168.1.0/24",
    "192.168.2.0/24",
    "192.168.3.0/24"
  ]
}
```

**Response (200 OK):**
```json
{
  "status": "success",
  "data": {
    "summary_route": "192.168.0.0/22",
    "range_covered": {
      "start_address": "192.168.0.0",
      "end_address": "192.168.3.255"
    },
    "routing_table_reduction": "75.0%"
  }
}
```

> Para agregar, la cantidad de redes debe ser potencia de 2 y la primera dirección debe ser múltiplo binario exacto del nuevo bloque.

## ⚡ Códigos de estado

| Código | Significado | Causa |
|---|---|---|
| `200` | OK | Cálculo procesado correctamente |
| `400` | Bad Request | Demanda excede capacidad del bloque, o redes no contiguas |
| `422` | Unprocessable Entity | Error de sintaxis (ej. octeto > 255, prefijo inválido) |

**Ejemplo de error 400:**
```json
{
  "status": "error",
  "error_code": "NETWORK_CAPACITY_EXCEEDED",
  "message": "La demanda total solicitada supera la capacidad del bloque base 192.168.1.0/24."
}
```

**Ejemplo de error 422:**
```json
{
  "detail": [
    {
      "loc": ["body", "base_network"],
      "msg": "El valor '192.168.1.300/24' no representa una dirección IPv4 válida.",
      "type": "value_error.ipv4_network"
    }
  ]
}
```

## 🎯 Alcance actual

- [x] Diseño de arquitectura de microservicio
- [x] Definición de contratos REST (request/response)
- [x] Validación matemática y analítica de FLSM/VLSM/Supernetting

## 👥 Equipo

Proyecto desarrollado para la Facultad de Ciencias e Ingenierías Físicas y Formales — Escuela Profesional de Ingeniería de Sistemas.

**Investigador principal:** Javier Fernando Angulo Osorio

**Desarrolladores:**
- Acosta Huamán Daniel Armando
- Benites Castro Arturo José
- Limache Quispe Felix Fabricio
- Mollo Huayhua Luis Felipe
- Vega Mamani Anthony Yerson

## 📚 Referencias

1. J. Postel, "Internet Protocol - DARPA Internet Program Protocol Specification," RFC 791, 1981.
2. V. Fuller and T. Li, "Classless Inter-domain Routing (CIDR)," RFC 4632, 2006.
3. F. Baker, "Requirements for IPv4 Routers," RFC 1812, 1995.
4. FastAPI — Documentación oficial: https://fastapi.tiangolo.com/
5. J. F. Kurose and K. W. Ross, *Computer Networking: A Top-Down Approach*, 8va ed. Pearson, 2021.
6. W. R. Stevens, *TCP/IP Illustrated, Volume 1*, 2da ed. Addison-Wesley, 2011.

## 📄 Licencia

_Por definir._
