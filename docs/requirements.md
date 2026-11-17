# Nivalis - Requisitos

## 1. Propósito

Nivalis es un sistema web interno para una heladería presencial. Permite
administrar productos, controlar el inventario compartido, registrar ventas
presenciales e identificar quién realizó cada operación.

Lo usan tres personas que trabajan en el mismo establecimiento. Los clientes
no acceden al sistema. El sistema registra operaciones, pero no procesa pagos.

## 2. Actores

| Rol    | Descripción                                                                      |
| ------ | -------------------------------------------------------------------------------- |
| SELLER | Operador del negocio. Realiza todas las tareas comerciales.                      |
| ADMIN  | Hace todo lo que hace un SELLER y además gestiona usuarios y listas controladas. |

### Permisos

| Acción                                                                | SELLER | ADMIN |
| --------------------------------------------------------------------- | ------ | ----- |
| Iniciar sesión y cambiar su propia contraseña                         | Sí     | Sí    |
| Registrar ventas                                                      | Sí     | Sí    |
| Consultar el historial de ventas (de todos los usuarios)              | Sí     | Sí    |
| Anular ventas (dentro de las 24 horas posteriores a su confirmación)  | Sí     | Sí    |
| Crear, editar y desactivar productos                                  | Sí     | Sí    |
| Registrar entradas y ajustes de inventario                            | Sí     | Sí    |
| Consultar movimientos de inventario                                   | Sí     | Sí    |
| Consultar reportes                                                    | Sí     | Sí    |
| Gestionar usuarios                                                    | No     | Sí    |
| Gestionar listas controladas (tipos, presentaciones, marcas, sabores) | No     | Sí    |

## 3. Requisitos funcionales

### Usuarios y acceso

- RF-01: El sistema permite iniciar y cerrar sesión con nombre de usuario y contraseña.
- RF-02: El ADMIN puede crear, editar y desactivar usuarios. Los usuarios no se eliminan físicamente.
- RF-03: El ADMIN puede restablecer la contraseña de cualquier usuario. Cada usuario puede cambiar la suya.
- RF-04: Un usuario desactivado no puede iniciar sesión.
- RF-05: El primer usuario ADMIN se crea al iniciar el sistema por primera vez, solo si no existe ningún usuario.

### Listas controladas

- RF-06: El ADMIN puede crear, editar y desactivar tipos de producto, presentaciones, marcas y sabores.
- RF-07: Cada sabor pertenece a una marca.
- RF-08: Los valores de las listas no se eliminan, solo se desactivan, para no afectar registros históricos.

### Productos

- RF-09: El SELLER y el ADMIN pueden registrar, editar, consultar y desactivar productos.
- RF-10: Un producto tiene como datos obligatorios: marca, sabor, nombre comercial, tipo, presentación y precio de lista.
- RF-11: Al registrar un producto, la marca se elige de una lista. Al elegir la marca, se habilitan solo los sabores de esa marca.
- RF-12: El sistema impide asociar a un producto un sabor que pertenezca a otra marca.
- RF-13: Un mismo helado en distintas presentaciones se registra como productos distintos.
- RF-14: Se puede buscar productos por nombre, marca, sabor, tipo y presentación.
- RF-15: Un producto desactivado no aparece para la venta, pero se conserva en el historial.

### Inventario

- RF-16: El SELLER y el ADMIN pueden registrar entradas de mercadería.
- RF-17: El SELLER y el ADMIN pueden registrar ajustes positivos y negativos. El motivo es obligatorio.
- RF-18: Todo cambio en el stock genera un movimiento de inventario con producto, tipo, cantidad, fecha y hora, usuario, motivo y referencia a la operación.
- RF-19: Los tipos de movimiento son: ENTRADA, VENTA, AJUSTE_POSITIVO, AJUSTE_NEGATIVO y REVERSION.
- RF-20: El sistema rechaza toda operación que deje el stock de un producto por debajo de 0.
- RF-21: Se puede consultar el historial de movimientos con filtros por producto, tipo, fecha y usuario.

### Ventas

- RF-22: El usuario puede armar un carrito: agregar productos, modificar cantidades, eliminar productos y ver subtotales y total.
- RF-23: Mientras la venta no se confirme, el inventario no se modifica.
- RF-24: Al vender, se puede indicar un precio distinto al de lista para cada línea. Puede ser cualquier valor mayor o igual a 0.
- RF-25: Un producto con stock 0 se muestra como agotado y no se puede vender hasta que se registre una entrada.
- RF-26: Al confirmar la venta, el sistema valida productos, cantidades y stock. Si algo falla, no se registra nada (ni la venta, ni los detalles, ni los movimientos).
- RF-27: Al confirmar la venta, se registra la venta, sus detalles, los movimientos de inventario y el descuento de stock como una sola operación.
- RF-28: Cada detalle de venta guarda el precio de lista vigente y el precio efectivamente cobrado en ese momento.
- RF-29: La venta registra al usuario que la realizó, la fecha y hora, y el total.
- RF-30: Al confirmar la venta se indica el método de pago informativo: EFECTIVO o YAPE.
- RF-31: Las ventas confirmadas no se editan. Si hubo un error, se anula y se registra una nueva.

### Anulaciones

- RF-32: El SELLER y el ADMIN pueden anular una venta confirmada dentro de las 24 horas posteriores a su confirmación. Pasado ese plazo, el sistema rechaza la anulación.
- RF-33: La anulación requiere un motivo y registra el usuario y la fecha.
- RF-34: La anulación genera movimientos de REVERSION y devuelve el stock.
- RF-35: Una venta no puede anularse más de una vez.
- RF-36: Las ventas nunca se eliminan físicamente.

### Historial y reportes

- RF-37: Se puede consultar el historial de ventas con filtros por fecha, usuario y estado. Cada venta muestra fecha y hora, usuario, productos, cantidades, total y estado.
- RF-38: El sistema ofrece estos reportes: ventas del día, ventas por rango de fechas, productos más vendidos, stock disponible, productos agotados, ventas por usuario y total vendido por usuario.
- RF-39: Los reportes muestran ingresos, no utilidad. No se calculan ganancias.

## 4. Requisitos no funcionales

- RNF-01 Integridad: las ventas y sus movimientos se guardan en una sola transacción. El stock nunca queda negativo, ni siquiera con ventas simultáneas de dos usuarios.
- RNF-02 Seguridad: las contraseñas se guardan con hash seguro. Las sesiones usan cookies seguras. La autorización se valida en el servidor, no solo en la interfaz.
- RNF-03 Seguridad: el servidor no confía en datos del navegador. Valida productos, cantidades y precios, y toma el usuario de la sesión.
- RNF-04 Seguridad: no se guardan contraseñas, claves ni secretos en el repositorio.
- RNF-05 Trazabilidad: toda operación registra su fecha y su usuario responsable.
- RNF-06 Usabilidad: la interfaz es clara y responsive, usable desde computadora y celular.
- RNF-07 Idioma: la interfaz está en español latino. El código, las tablas y los nombres internos están en inglés.
- RNF-08 Moneda: todos los montos están en soles (S/), con dos decimales.
- RNF-09 Zona horaria: el sistema opera en America/Lima.
- RNF-10 Mantenibilidad: los cambios en la base de datos se gestionan con migraciones versionadas.
- RNF-11 Pruebas: las reglas críticas (stock, transacciones, permisos, anulaciones) cuentan con pruebas automáticas.
- RNF-12 Operación: el sistema se despliega en un servidor propio sin Docker, con copias de seguridad automáticas almacenadas fuera del servidor.

## 5. Reglas de negocio

- RN-01: El stock de un producto nunca es negativo.
- RN-02: Las cantidades son siempre números enteros mayores que 0.
- RN-03: El stock inicial de un producto nuevo es 0. Aumenta mediante una entrada.
- RN-04: El stock solo cambia mediante un movimiento de inventario.
- RN-05: Las ventas, los movimientos, los usuarios y los productos no se eliminan físicamente.
- RN-06: Una venta con precio 0 cuenta como venta y como unidades salidas, pero suma 0 a los ingresos.
- RN-07: El precio de una venta pasada nunca se recalcula con el precio actual del producto.
- RN-08: El método de pago es solo informativo. El sistema no procesa pagos.
- RN-09: No existe stock mínimo. Con stock 0 el producto está agotado.
- RN-10: El plazo de anulación se cuenta desde la fecha y hora de confirmación de la venta.

## 6. Fuera del MVP

- Tienda virtual, pedidos online y carrito para clientes.
- Procesamiento de pagos e integraciones bancarias o financieras.
- Varios negocios o empresas en un mismo sistema.
- Microservicios.
- Venta sin conexión con sincronización.
- Cálculo de utilidad o ganancias.
- Recuperación de contraseña por correo.
- Inteligencia artificial.
