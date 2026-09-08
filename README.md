# Calculadora Web

Calculadora de operaciones aritméticas básicas que se ejecuta enteramente en el
navegador. Sin servidor, sin cuentas de usuario y sin llamadas de red: se abre
el fichero y funciona.

Este repositorio es el laboratorio donde se construye. Lo que hay aquí —contexto,
decisiones y criterios— es la fuente de verdad del proyecto, y está escrito para
que cualquiera que llegue nuevo entienda **por qué** las cosas son como son, no
sólo qué hacen.

## El problema

Los formularios internos de captura de datos obligan hoy a salir de la aplicación
para hacer una cuenta rápida: abrir la calculadora del sistema operativo, calcular,
volver y transcribir. Ese ir y venir es la fuente de la mayoría de los errores de
transcripción que se detectan en revisión.

La calculadora se incrustará más adelante en esos formularios como componente. Por
eso el motor de cálculo se construye desde el principio desacoplado de la interfaz:
lo que se va a reutilizar es el motor, no la pantalla.

## Alcance

### Incluido

- Suma, resta, multiplicación y división sobre dos operandos.
- Encadenamiento de operaciones sin pulsar el signo igual entre ellas.
- Entrada por teclado físico y por los botones de la pantalla.
- Borrado de la entrada actual (CE) y reinicio total del estado (C).
- Tratamiento explícito de la división por cero.
- Interfaz utilizable desde 320 px de ancho hasta escritorio.

### Excluido

Lo siguiente queda fuera de forma deliberada. Son las peticiones que aparecen a
mitad de un desarrollo de este tipo, y dejarlas escritas evita discutirlas dos veces:

- Funciones científicas: trigonometría, logaritmos, potencias y raíces.
- Historial de operaciones y teclas de memoria (M+, M-, MR).
- Cuentas de usuario, sincronización o almacenamiento en servidor.
- Internacionalización más allá del castellano de la interfaz inicial.

Cualquier añadido sobre esta lista se replanifica; no entra por la puerta de atrás.

## Decisiones ya tomadas

Estas cuatro decisiones están cerradas. Se documentan aquí porque son las que un
recién llegado tiende a cuestionar, y la respuesta merece estar escrita una vez.

### El redondeo es a 10 decimales significativos

En coma flotante, `0.1 + 0.2` da `0.30000000000000004`. Es correcto según IEEE 754
y es inaceptable en pantalla. Todo resultado se redondea a 10 decimales
significativos **en la frontera de presentación**, nunca dentro del motor: el motor
devuelve el número tal cual y quien lo muestra decide cómo.

### La división por cero devuelve un error, no infinito

JavaScript devuelve `Infinity` al dividir entre cero, sin lanzar excepción. El motor
no propaga ese valor: devuelve un resultado de tipo error, con mensaje legible. La
calculadora queda en estado de error y se recupera con la tecla C. Nunca se muestra
`Infinity` ni `NaN`.

### No hay framework de interfaz

Diez botones y una pantalla no justifican React ni su tamaño de descarga. La
interfaz es HTML y CSS. Esta decisión se revisa sólo si la calculadora pasa a
componerse con otros componentes que ya usen framework.

### No hay capa de datos

No se almacena nada. Sin historial y sin memoria, no hay estado que persistir. Toda
propuesta que introduzca una base de datos parte de un requisito que no existe.

## Arquitectura

Tres capas, con una única dirección de dependencia:

| Capa | Responsabilidad | Depende de |
|---|---|---|
| Presentación | Dibuja pantalla y botones. Traduce pulsaciones de ratón y teclado a eventos de dominio. | Estado |
| Estado | Máquina de estados: operando actual, operando previo, operación pendiente y bandera de error. | Motor |
| Motor | Funciones puras: reciben dos números, devuelven resultado o error. Sin DOM, sin efectos. | — |

El motor no conoce nada. Esa es la razón de separarlo: es la parte que hay que
probar de verdad, y probarla a través del DOM la vuelve lenta y frágil. Como
función pura se prueba en milisegundos.

### Modelo de estado

El estado es un objeto de cinco campos: `displayValue`, `previousValue`,
`pendingOperator`, `waitingForOperand` y `errorState`. Cada transición es una
función que recibe el estado y un evento, y devuelve un estado nuevo.

Esta forma elimina el defecto más común en calculadoras —la pulsación que se
interpreta según lo que quedó en pantalla— porque el evento nunca lee el DOM.

## Stack

| Componente | Elección |
|---|---|
| Lenguaje | TypeScript |
| Interfaz | HTML + CSS, sin framework |
| Build | Vite |
| Pruebas unitarias | Vitest |
| Pruebas de extremo a extremo | Playwright |
| Calidad | ESLint + Prettier |
| Despliegue | Hosting estático |

## Requisitos no funcionales

- **Rendimiento**: carga por debajo de 1 s en 3G simulada; respuesta a una
  pulsación por debajo de 100 ms.
- **Compatibilidad**: últimas dos versiones estables de Chrome, Firefox, Safari y Edge.
- **Accesibilidad**: WCAG 2.1 nivel AA. Navegación completa por teclado, foco
  visible, contraste mínimo 4,5:1 y etiquetas ARIA en cada botón.
- **Cobertura**: el motor de cálculo se mantiene igual o por encima del 90 % de
  líneas cubiertas.

La accesibilidad se verifica desde el primer día con axe-core en integración
continua. Dejarla para el final es la forma conocida de no hacerla.

## Cómo se prueba

Una prueba se considera válida cuando **falla al eliminar el código que la
sostiene**. Una prueba que pasa contra un motor vacío no prueba nada.

Casos que no pueden faltar:

- Dividir entre cero muestra error, y la siguiente C devuelve la calculadora a un
  estado usable.
- `0.1 + 0.2` muestra `0.3`.
- Pulsar dos operadores seguidos aplica el último, sin resultado intermedio espurio.
- Pulsar `=` sin operación pendiente no altera la pantalla.
- Encadenar `2 + 3 + 4` muestra `9` sin pulsar `=` hasta el final.
- La aplicación es operable de principio a fin sin tocar el ratón.

## Orden de construcción

El motor y el estado se construyen y se prueban **antes** de que exista un solo
botón. Invertir el orden obliga a probar aritmética a través de la interfaz, que
es lento de escribir y frágil de mantener.

1. Preparación: repositorio, Vite, TypeScript, ESLint, Vitest e integración continua.
2. Motor: las cuatro operaciones, el redondeo y el tratamiento de error.
3. Estado: máquina de estados y encadenamiento.
4. Interfaz: maquetación, responsive, teclado y accesibilidad.
5. Cierre: pruebas de extremo a extremo, revisión de accesibilidad y despliegue.

## Glosario

- **Operando**: cada uno de los dos números que participan en una operación.
- **Operación pendiente**: operador introducido cuyo segundo operando aún no se ha
  completado.
- **Estado de error**: situación en la que la última operación no produjo un número
  válido. Sólo se sale de él con C.
- **Frontera de presentación**: el punto donde un valor del motor se convierte en
  texto en pantalla. Es donde ocurre el redondeo.
