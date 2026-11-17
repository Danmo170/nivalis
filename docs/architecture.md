# Nivalis - Arquitectura

Este documento describe cómo está organizado Nivalis y por qué. El qué
se construye está en `requirements.md`.

## 1. Visión general

Nivalis es un monolito modular: una sola aplicación Spring Boot que sirve
las vistas HTML (Thymeleaf) y se conecta a una base de datos MySQL.

```
Navegador --HTTPS--> Nginx --HTTP--> Spring Boot --JDBC--> MySQL
```

Se eligió un monolito porque lo usan tres personas, lo desarrolla una
sola persona y un solo despliegue es más simple de operar y mantener.
Los módulos separados por paquetes dejan abierta la posibilidad de
dividirlo en el futuro si hiciera falta.

## 2. Tecnologías

| Componente    | Tecnología                                               |
| ------------- | -------------------------------------------------------- |
| Lenguaje      | Java 25                                                  |
| Framework     | Spring Boot (versión estable compatible con Java 25)     |
| Persistencia  | Spring Data JPA con Hibernate                            |
| Seguridad     | Spring Security con sesiones                             |
| Validación    | Bean Validation                                          |
| Base de datos | MySQL 8.4 LTS (InnoDB, utf8mb4)                          |
| Migraciones   | Flyway                                                   |
| Vistas        | Thymeleaf, HTML, CSS, JavaScript, Bootstrap 5            |
| Construcción  | Maven                                                    |
| Producción    | Ubuntu 24.04, Nginx, systemd, Let's Encrypt (sin Docker) |

## 3. Organización del código

El código se organiza por módulo funcional, no por capa técnica. Lo que
cambia junto vive junto.

| Módulo    | Responsabilidad                                                         |
| --------- | ----------------------------------------------------------------------- |
| config    | Configuración general de la aplicación                                  |
| security  | Autenticación, autorización y roles                                     |
| user      | Usuarios del sistema                                                    |
| product   | Productos y listas controladas (tipos, presentaciones, marcas, sabores) |
| inventory | Stock y movimientos de inventario                                       |
| sale      | Ventas, detalles y anulaciones                                          |
| report    | Consultas de reportes (solo lectura)                                    |

### Capas dentro de cada módulo

| Capa       | Responsabilidad                                                                     |
| ---------- | ----------------------------------------------------------------------------------- |
| Controller | Recibe solicitudes HTTP, valida el formato y elige la vista. Sin reglas de negocio. |
| Service    | Contiene las reglas de negocio y define las transacciones.                          |
| Repository | Acceso y persistencia de datos.                                                     |
| Entity     | Representa las tablas de la base de datos.                                          |
| DTO        | Datos que entran y salen hacia la vista. Las entidades no se exponen directamente.  |

### Dependencias entre módulos

- `sale` depende de `product` e `inventory`.
- `inventory` depende de `product` y `user`.
- `report` solo lee datos de los demás.
- No se permiten dependencias circulares.

## 4. Decisiones de arquitectura

| #   | Decisión                                                                                                                | Motivo                                                                                     |
| --- | ----------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| 1   | Solo `inventory` modifica el stock. `sale` le pide registrar un movimiento y nunca toca el stock directamente.          | Un único punto de entrada garantiza que no haya stock sin movimiento (RN-04).              |
| 2   | El carrito vive en el navegador (JavaScript). Al confirmar se envía el contenido completo.                              | Es simple, no usa estado en el servidor y encaja con RF-23.                                |
| 3   | El servidor no confía en el navegador: revalida productos, cantidades, precios y stock, y toma el usuario de la sesión. | Seguridad e integridad (RNF-03).                                                           |
| 4   | Autenticación con sesiones y cookies seguras. Sin JWT.                                                                  | Con vistas del lado del servidor, las sesiones son más simples y adecuadas.                |
| 5   | Roles con jerarquía: ADMIN incluye todo lo de SELLER.                                                                   | Refleja la matriz de permisos y evita duplicar reglas.                                     |
| 6   | Dinero en DECIMAL (MySQL) y BigDecimal (Java).                                                                          | Evitar errores de redondeo.                                                                |
| 7   | Listas controladas (tipo, presentación, marca, sabor) como tablas gestionadas por el ADMIN.                             | Permiten ampliar valores sin modificar código y garantizan integridad con claves foráneas. |
| 8   | Nada se elimina físicamente: se desactiva o se anula.                                                                   | Trazabilidad e integridad del historial (RN-05).                                           |
| 9   | Zona horaria `America/Lima` definida de forma explícita.                                                                | Los reportes "del día" dependen de ella.                                                   |

## 5. Modelo de datos (resumen)

Nueve tablas: cinco del núcleo y cuatro listas controladas. El detalle
de columnas, tipos, restricciones e índices se documentará en
`database-design.md` antes de crear las entidades.

```
BRAND 1 ---< FLAVOR
PRODUCT_TYPE 1 ---< PRODUCT >--- 1 PRESENTATION
FLAVOR 1 ---< PRODUCT
USER 1 ---< SALE 1 ---< SALE_DETAIL >--- 1 PRODUCT
USER 1 ---< INVENTORY_MOVEMENT >--- 1 PRODUCT
SALE 1 ---< INVENTORY_MOVEMENT   (referencia opcional)
```

| Tabla                                     | Responsabilidad                                         |
| ----------------------------------------- | ------------------------------------------------------- |
| user                                      | Cuentas del sistema                                     |
| product                                   | Catálogo, precio de lista y stock actual                |
| sale                                      | Cabecera de la venta, estado y datos de anulación       |
| sale_detail                               | Líneas de la venta con precio de lista y precio cobrado |
| inventory_movement                        | Libro mayor del stock                                   |
| product_type, presentation, brand, flavor | Listas controladas                                      |

Puntos a resolver en la Fase 3:

- Cómo garantizar que el sabor de un producto pertenezca a su marca
  (RF-12): en el servicio, en la base de datos, o en ambos.
- Si `product` guarda la marca directamente o la deduce del sabor.

## 6. Flujo de venta

1. El usuario inicia sesión y busca productos.
2. Arma el carrito en el navegador. Los totales son solo ayuda visual.
3. Elige el método de pago (EFECTIVO o YAPE) y confirma.
4. El navegador envía solo producto, cantidad y precio indicado por línea.
5. El servidor, en una sola transacción:
   1. Identifica al usuario desde la sesión.
   2. Carga los productos y verifica que existan y estén activos.
   3. Valida cantidades (enteras y mayores que 0) y precios (mayores o iguales a 0).
   4. Bloquea las filas de los productos, ordenadas por id para evitar interbloqueos.
   5. Verifica stock suficiente.
   6. Crea la venta y sus detalles con precio de lista y precio cobrado.
   7. Registra los movimientos VENTA y descuenta el stock mediante `inventory`.
   8. Calcula el total y confirma la transacción.
6. Si algo falla, se revierte todo y se informa el motivo.

## 7. Flujo de anulación

1. El usuario abre una venta confirmada e indica el motivo.
2. En una sola transacción el servidor verifica que:
   - la venta esté en estado CONFIRMED,
   - no hayan pasado más de 24 horas desde su confirmación.
3. Marca la venta como CANCELLED con usuario, fecha y motivo.
4. Registra los movimientos REVERSION y restituye el stock mediante `inventory`.

## 8. Riesgos principales

| Riesgo                                             | Mitigación prevista                                          |
| -------------------------------------------------- | ------------------------------------------------------------ |
| Dos ventas simultáneas sobre la última unidad      | Bloqueo pesimista de filas y prueba que lo demuestre         |
| Doble envío de una venta (doble clic, reintentos)  | Botón deshabilitado y token de idempotencia en el servidor   |
| Stock desincronizado respecto a los movimientos    | Punto único de modificación y consulta de verificación       |
| Reportes "del día" mal calculados por zona horaria | `America/Lima` definida en la aplicación y en la conexión    |
| Autorización solo en la interfaz                   | Reglas en Spring Security y pruebas por rol                  |
| Pérdida de datos                                   | Backups automáticos con copia externa y restauración probada |
| Secretos en el repositorio                         | Variables de entorno y `.gitignore` desde el primer commit   |
| Pruebas contra una base distinta de la real        | Probar contra MySQL real                                     |

## 9. Decisiones por documentar como ADR

- Sesiones en lugar de JWT.
- Carrito en el navegador.
- Bloqueo pesimista para el control de concurrencia.
- Listas controladas como tablas.
- Creación del primer ADMIN desde variable de entorno.


