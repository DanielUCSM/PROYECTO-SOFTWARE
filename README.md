# NetSegment API

Microservicio para segmentacion, asignacion y agregacion de direccionamiento IPv4 mediante FLSM, VLSM y Supernetting (CIDR).

[![Estado](https://img.shields.io/badge/estado-en%20desarrollo-yellow)]()
[![TRL](https://img.shields.io/badge/TRL-3%20(prueba%20de%20concepto)-blue)]()
[![Python](https://img.shields.io/badge/python-3.11%2B-blue)]()
[![Framework](https://img.shields.io/badge/framework-FastAPI-009688)]()

## Estado del proyecto

Este proyecto se encuentra actualmente en TRL 3 (prueba de concepto a nivel analitico y de diseno). Esto significa que:

- La arquitectura, los esquemas de datos y la logica algoritmica estan completamente disenados y validados analiticamente.
- El contrato de API (endpoints, formatos de request/response, codigos de error) esta definido y documentado.

Este README describe ese diseno: la arquitectura del microservicio y el contrato de API tal como fueron validados analiticamente.

## Descripcion

En la gestion de redes, calcular subredes manualmente (hojas de calculo, calculadoras web) genera errores frecuentes: solapamiento de direcciones (IP overlap), desperdicio de bloques IPv4 y sobrecarga de tablas de enrutamiento por falta de sumarizacion.

NetSegment API resuelve esto exponiendo un microservicio REST que automatiza:

- FLSM (Fixed Length Subnet Mask) — division de una red en subredes de igual tamano.
- VLSM (Variable Length Subnet Mask) — asignacion optima de bloques segun demanda real de hosts, usando un algoritmo voraz (greedy).
- Supernetting / CIDR — agregacion de rutas contiguas en una sola entrada de enrutamiento.

A diferencia de las calculadoras web tradicionales, esta pensado para ser consumido por otro software: pipelines de CI/CD, scripts de Ansible/Terraform, o cualquier sistema de aprovisionamiento de infraestructura (IaC).

## Arquitectura

```
Cliente HTTP (cURL / Postman / Script IaC / Frontend)
        |  POST (JSON)
        v
Servidor ASGI (Uvicorn, TCP:8000)
        |
        v
Capa de enrutamiento (FastAPI Router)
        |
        v
Validacion de esquemas (Pydantic v2) --X--> 422 Unprocessable Entity
        | OK
        v
Selector de operacion (/flsm | /vlsm | /aggregate)
        |
        v
Motor logico de red (modulo ipaddress, algebra de Boole)
        |
        v
Validacion de limites y capacidad --X--> 400 Bad Request
        | OK
        v
Respuesta 200 OK (JSON con detalle de red)
```

## Stack tecnologico

| Componente | Tecnologia |
|---|---|
| Lenguaje | Python 3.11+ |
| Framework web | FastAPI (ASGI) |
| Validacion de datos | Pydantic v2 |
| Calculo de direcciones | Modulo nativo `ipaddress` |
| Servidor | Uvicorn (uvloop + httptools) |
| Aislamiento de dependencias | `venv` |

## Requisitos

### Minimos
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

## Contrato de API (diseno validado)

Todos los endpoints operan bajo el prefijo `/api/v1`, con metodo `POST` y JSON en UTF-8.

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

El algoritmo reordena automaticamente las demandas de mayor a menor antes de asignar los bloques, para evitar fragmentacion.

### 3. Supernetting — Agregacion de rutas CIDR

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

Para agregar, la cantidad de redes debe ser potencia de 2 y la primera direccion debe ser multiplo binario exacto del nuevo bloque.

## Codigos de estado

| Codigo | Significado | Causa |
|---|---|---|
| `200` | OK | Calculo procesado correctamente |
| `400` | Bad Request | Demanda excede capacidad del bloque, o redes no contiguas |
| `422` | Unprocessable Entity | Error de sintaxis (ej. octeto > 255, prefijo invalido) |

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
      "msg": "El valor '192.168.1.300/24' no representa una direccion IPv4 valida.",
      "type": "value_error.ipv4_network"
    }
  ]
}
```

## Alcance actual

- [x] Diseno de arquitectura de microservicio
- [x] Definicion de contratos REST (request/response)
- [x] Validacion matematica y analitica de FLSM/VLSM/Supernetting

## Equipo

Proyecto desarrollado para la Facultad de Ciencias e Ingenierias Fisicas y Formales — Escuela Profesional de Ingenieria de Sistemas.

**Investigador principal:** Javier Fernando Angulo Osorio

**Desarrolladores:**
- Acosta Huaman Daniel Armando
- Benites Castro Arturo Jose
- Limache Quispe Felix Fabricio
- Mollo Huayhua Luis Felipe
- Vega Mamani Anthony Yerson

## Referencias

1. J. Postel, "Internet Protocol - DARPA Internet Program Protocol Specification," RFC 791, 1981.
2. V. Fuller and T. Li, "Classless Inter-domain Routing (CIDR)," RFC 4632, 2006.
3. F. Baker, "Requirements for IPv4 Routers," RFC 1812, 1995.
4. FastAPI — Documentacion oficial: https://fastapi.tiangolo.com/
5. J. F. Kurose and K. W. Ross, Computer Networking: A Top-Down Approach, 8va ed. Pearson, 2021.
6. W. R. Stevens, TCP/IP Illustrated, Volume 1, 2da ed. Addison-Wesley, 2011.

## Licencia

Por definir.
