# Actividad Neo4j — Detección de Fraude Bancario

**Unidad Curricular:** Base de Datos III  
**Eje III:** El ecosistema NoSQL  
**Apunte de Cátedra 16:** Introducción a Neo4j y el Paradigma de Grafos

---

## Contexto del escenario

Somos arquitectos de datos para un banco. Se sospecha de una **red de fraude** donde distintos usuarios comparten números de teléfono e IPs, pero registran **DNIs diferentes**.

El modelo relacional original tiene estas tablas:

| Tabla | Campos principales |
|-------|-------------------|
| `USUARIOS` | `id` (PK), `dni` (UK), `nombre` |
| `CUENTAS` | `id` (PK), `usuario_id` (FK), `numero_cuenta` |
| `TELEFONOS` | `id` (PK), `usuario_id` (FK), `numero` |
| `CONEXIONES_IP` | `id` (PK), `usuario_id` (FK), `ip_address`, `fecha` |
| `TRANSFERENCIAS` | `id` (PK), `cuenta_origen_id` (FK), `cuenta_destino`, `monto`, `fecha` |

**Relaciones SQL:**
- Un usuario **posee** varias cuentas
- Un usuario **registra** varios teléfonos
- Un usuario **genera** varias conexiones IP
- Una cuenta **realiza** varias transferencias

---

## Actividad 1 — Diseño del modelo en Neo4j

### Idea central

En SQL, la IP y el teléfono son **atributos** de filas distintas. Para detectar fraude hay que hacer múltiples `JOIN`.

En Neo4j, la IP y el teléfono se modelan como **nodos compartidos**. Si dos usuarios distintos apuntan al mismo nodo `:IP` o `:Telefono`, la conexión fraudulenta queda visible de inmediato.

### Nodos

| Etiqueta | Propiedades | Origen SQL |
|----------|-------------|------------|
| `:Usuario` | `id`, `dni`, `nombre` | Tabla `USUARIOS` |
| `:Cuenta` | `id`, `numero_cuenta` | Tabla `CUENTAS` |
| `:Telefono` | `numero` | Campo `TELEFONOS.numero` |
| `:IP` | `direccion` | Campo `CONEXIONES_IP.ip_address` |

### Relaciones

| Tipo | Desde → Hacia | Propiedades | Equivalente SQL |
|------|---------------|-------------|-----------------|
| `:POSEE` | Usuario → Cuenta | — | `CUENTAS.usuario_id` |
| `:REGISTRA` | Usuario → Teléfono | — | `TELEFONOS.usuario_id` |
| `:CONECTA_DESDE` | Usuario → IP | `fecha` | `CONEXIONES_IP` |
| `:TRANSFIERE_A` | Cuenta → Cuenta | `monto`, `fecha` | `TRANSFERENCIAS` |

> La cuenta destino (`cuenta_destino` en SQL) también se modela como nodo `:Cuenta`, identificada por `numero_cuenta`. Así, dos transferencias hacia la misma cuenta comparten el mismo nodo destino.

### Diagrama del grafo

```
                    ┌─────────────┐
                    │   :Usuario │
                    │ dni, nombre│
                    └──────┬──────┘
           ┌───────────────┼───────────────┐
           │ POSEE         │ REGISTRA      │ CONECTA_DESDE {fecha}
           ▼               ▼               ▼
    ┌──────────┐    ┌───────────┐    ┌──────────┐
    │  :Cuenta │    │ :Telefono │    │   :IP    │
    │  numero  │    │  numero   │    │ direccion│
    └────┬─────┘    └───────────┘    └──────────┘
         │ TRANSFIERE_A {monto, fecha}
         ▼
    ┌──────────┐
    │  :Cuenta │  ← cuenta destino
    │  numero  │
    └──────────┘
```

### Ejemplo de patrón de fraude

```
(Ana:Usuario)──POSEE──>(Cuenta 001)
(Ana)──REGISTRA──>(Tel +54 351 555-1234)<──REGISTRA──(Carlos:Usuario)
(Ana)──CONECTA_DESDE──>(IP 192.168.1.50)<──CONECTA_DESDE──(Carlos)
(Cuenta 001)──TRANSFIERE_A──>(Cuenta 999-0001)<──TRANSFIERE_A──(Cuenta 009)
```

**Ana y Carlos:**
- Tienen DNI distintos
- Comparten el mismo teléfono
- Comparten la misma IP
- Transfieren a la misma cuenta destino

### Comparación SQL vs Grafo

**SQL** — detectar usuarios con la misma IP:

```sql
SELECT u1.dni, u2.dni, c1.ip_address
FROM USUARIOS u1
JOIN CONEXIONES_IP c1 ON u1.id = c1.usuario_id
JOIN CONEXIONES_IP c2 ON c1.ip_address = c2.ip_address
JOIN USUARIOS u2 ON c2.usuario_id = u2.id
WHERE u1.id <> u2.id;
```

**Cypher** — mismo patrón, sin JOINs:

```cypher
MATCH (u1:Usuario)-[:CONECTA_DESDE]->(ip:IP)<-[:CONECTA_DESDE]-(u2:Usuario)
WHERE u1.dni <> u2.dni
RETURN u1.dni, u2.dni, ip.direccion
```

---

## Actividad 2 — Pseudoconsulta Cypher

### Objetivo

Encontrar **dos personas distintas** que:
1. Se conectan desde la **misma IP**
2. Enviaron dinero a la **misma cuenta**

### Consulta principal

```cypher
// Paso 1: Dos usuarios distintos que comparten la misma IP
MATCH (u1:Usuario)-[:CONECTA_DESDE]->(ip:IP)<-[:CONECTA_DESDE]-(u2:Usuario)
WHERE u1.dni <> u2.dni
  AND id(u1) < id(u2)

// Paso 2: Ambos transfirieron a la misma cuenta destino
MATCH (u1)-[:POSEE]->(:Cuenta)-[:TRANSFIERE_A]->(destino:Cuenta)
MATCH (u2)-[:POSEE]->(:Cuenta)-[:TRANSFIERE_A]->(destino)

// Paso 3: Resultado
RETURN u1.dni              AS persona_1,
       u1.nombre           AS nombre_1,
       u2.dni              AS persona_2,
       u2.nombre           AS nombre_2,
       ip.direccion        AS ip_compartida,
       destino.numero_cuenta AS cuenta_destino_comun
```

### Explicación de cada parte

| Fragmento | Qué hace |
|-----------|----------|
| `(u1)-[:CONECTA_DESDE]->(ip)<-[:CONECTA_DESDE]-(u2)` | Encuentra dos usuarios conectados al mismo nodo IP |
| `WHERE u1.dni <> u2.dni` | Garantiza que son personas distintas |
| `id(u1) < id(u2)` | Evita resultados duplicados (Ana-Carlos y Carlos-Ana) |
| `-[:TRANSFIERE_A]->(destino)` | Verifica que ambos enviaron dinero al mismo nodo cuenta |
| `RETURN ...` | Devuelve el patrón sospechoso completo |

### Variante alternativa

Si `cuenta_destino` se guarda como propiedad en la relación en lugar de un nodo:

```cypher
MATCH (u1:Usuario)-[:CONECTA_DESDE]->(ip:IP)<-[:CONECTA_DESDE]-(u2:Usuario)
WHERE u1.dni <> u2.dni AND id(u1) < id(u2)

MATCH (u1)-[:POSEE]->(:Cuenta)-[t1:TRANSFIERE_A]->()
MATCH (u2)-[:POSEE]->(:Cuenta)-[t2:TRANSFIERE_A]->()
WHERE t1.cuenta_destino = t2.cuenta_destino

RETURN u1.dni, u2.dni, ip.direccion, t1.cuenta_destino
```

---

## Datos de prueba para Neo4j Aura

Copiar y ejecutar en la consola de [Neo4j Aura](https://console.neo4j.io/):

```cypher
// Usuarios
CREATE (u1:Usuario {id: 1, dni: "30111222", nombre: "Ana"})
CREATE (u2:Usuario {id: 2, dni: "28444555", nombre: "Carlos"})

// Cuentas
CREATE (c1:Cuenta {id: 1, numero_cuenta: "001-2345"})
CREATE (c2:Cuenta {id: 2, numero_cuenta: "009-8877"})
CREATE (cd:Cuenta {id: 3, numero_cuenta: "999-0001"})

// Nodos compartidos (señal de fraude)
CREATE (ip:IP {direccion: "192.168.1.50"})
CREATE (tel:Telefono {numero: "+54 351 555-1234"})

// Relaciones
CREATE (u1)-[:POSEE]->(c1)
CREATE (u2)-[:POSEE]->(c2)
CREATE (u1)-[:REGISTRA]->(tel)
CREATE (u2)-[:REGISTRA]->(tel)
CREATE (u1)-[:CONECTA_DESDE {fecha: datetime("2024-03-15T10:00:00")}]->(ip)
CREATE (u2)-[:CONECTA_DESDE {fecha: datetime("2024-03-16T14:30:00")}]->(ip)
CREATE (c1)-[:TRANSFIERE_A {monto: 5000.00, fecha: datetime("2024-03-17T09:00:00")}]->(cd)
CREATE (c2)-[:TRANSFIERE_A {monto: 8000.00, fecha: datetime("2024-03-18T11:00:00")}]->(cd)
```

Después de cargar los datos, ejecutar la consulta de la Actividad 2. El resultado esperado es:

| persona_1 | nombre_1 | persona_2 | nombre_2 | ip_compartida | cuenta_destino_comun |
|-----------|----------|-----------|----------|---------------|----------------------|
| 30111222  | Ana      | 28444555  | Carlos   | 192.168.1.50  | 999-0001             |

---

## Resumen

| Concepto | Respuesta |
|----------|-----------|
| Nodos | `:Usuario`, `:Cuenta`, `:Telefono`, `:IP` |
| Propiedades | `dni`, `nombre`, `numero_cuenta`, `numero`, `direccion`, `monto`, `fecha` |
| Relaciones | `:POSEE`, `:REGISTRA`, `:CONECTA_DESDE`, `:TRANSFIERE_A` |
| ¿Por qué IP/Teléfono como nodos? | Conectan usuarios distintos a través del mismo punto y detectan fraude sin JOINs costosos |

---

## Referencias

- [Neo4j Aura Console](https://console.neo4j.io/)
- Apunte de Cátedra 16 — Base de Datos III
- Guía Práctica: Dominando Neo4j Aura
