# Explicación del Código Fuente – Módulo de Noticias

## Introducción

Dentro del proyecto *Escucha tu Historia – Audioguía de Martos*, me encargué del desarrollo completo del módulo de noticias, así como de las traducciones de la aplicación y de la persistencia local de datos en disco. A continuación se describe en detalle cada componente desarrollado.

---

## 1. Modelo de datos – `noticia_model.dart`

Este archivo define la clase `Noticia`, que representa la entidad principal del módulo. Contiene los siguientes campos:

- `id`: identificador único de la noticia (UUID en formato `String`).
- `titulo` y `subtitulo`: textos principales de presentación.
- `contenido`: cuerpo completo de la noticia.
- `fechaPublicacion`: fecha opcional de publicación oficial.
- `imagenUrl`: URL opcional de la imagen de cabecera.
- `createdAt` y `lastModified`: fechas de auditoría gestionadas por el backend.

La clase implementa tres métodos estándar de serialización:

- `fromJson`: factory constructor que construye una instancia a partir de un `Map<String, dynamic>`, parseando los campos de fecha con `DateTime.parse`.
- `toJson`: convierte la instancia a mapa para su posible envío o almacenamiento.
- `copyWith`: permite crear una copia del objeto modificando solo los campos deseados, siguiendo el patrón inmutable habitual en Flutter.

---

## 2. Servicio – `noticias_service.dart`

La clase `NoticiasService` centraliza toda la lógica de acceso a datos del módulo, tanto remota como local.

### Comunicación con el backend

El método `fetchNoticias` realiza una petición HTTP GET al endpoint `/api/v1/public/news` del backend propio del proyecto. Si la respuesta es exitosa (código 200), deserializa el JSON en una lista de objetos `Noticia` y la ordena por `createdAt` de más reciente a más antigua. Cualquier error de red o de estado HTTP lanza una excepción que el provider capturará para mostrársela al usuario.

Se incluyó también el método `fetchNoticiasMock` junto con `obtenerNoticiasMock`, que devuelve una lista de noticias estáticas de ejemplo. Esto permitió desarrollar y probar la interfaz sin depender del backend.

### Persistencia local con `SharedPreferences`

Para recordar qué noticias ha leído el usuario entre sesiones, el servicio guarda en disco un conjunto de identificadores usando `SharedPreferences`, bajo la clave `noticias_leidas_ids`.

Los métodos implementados son:

- `cargarIdsLeidas`: lee la lista almacenada y la convierte en un `Set<String>`.
- `guardarIdsLeidas`: persiste el conjunto actualizado de IDs.
- `marcarComoLeida`: añade un ID concreto al conjunto y guarda.
- `marcarTodasComoLeidas`: une todos los IDs de la lista de noticias al conjunto y guarda.
- `limpiarIdsLeidas`: elimina la clave de `SharedPreferences`, útil para pruebas y depuración.

---

## 3. Provider – `noticias_provider.dart`

`NoticiasProvider` extiende `ChangeNotifier` y actúa como capa de estado reactiva siguiendo el patrón Provider, el elegido en el proyecto para la gestión de estado.

### Estado gestionado

- `todasLasNoticias`: lista completa cargada desde el servicio.
- `idsLeidas`: conjunto de IDs de noticias leídas, recuperado de disco al inicializar.
- `cargando`: booleano que indica si hay una operación asíncrona en curso.
- `error`: mensaje de error en caso de fallo, o `null` si todo va bien.

### Getters derivados

- `noticiasNoLeidas`: filtra la lista completa devolviendo solo las no leídas.
- `cantidadNoLeidas`: número de noticias no leídas, usado para la insignia del icono de notificaciones.
- `estaLeida(id)`: comprueba si una noticia concreta ya fue leída.

### Ciclo de vida y métodos

En el constructor se llama a `cargarNoticias` para obtener los datos del servidor y a `_inicializar` para recuperar de disco las IDs leídas de sesiones anteriores. Ambas son operaciones asíncronas independientes que notifican a los widgets cuando terminan.

`marcarComoLeida` y `marcarTodasComoLeidas` delegan la escritura en disco al servicio y luego actualizan el estado en memoria para que la UI se refresque inmediatamente sin necesidad de recargar del servidor. El método `limpiar` se implementó como utilidad de desarrollo para resetear el estado completo.

---

## 4. Rutas – `app_router.dart`

La navegación de la aplicación se gestiona con el paquete `go_router`. Se definieron dos rutas propias del módulo dentro del `GoRouter` global:

- `/noticias` → carga `NoticiasScreen`, la pantalla del listado.
- `/noticia/:id` → carga `DetalleNoticiaScreen`, extrayendo el parámetro `id` de la URL con `state.pathParameters['id']`.

Esta estructura de rutas con parámetros permite navegar directamente a cualquier noticia conociendo solo su ID, tanto desde el listado como desde el panel de notificaciones del `HomeScreen`.

---

## 5. Pantalla de listado – `noticias_screen.dart`

`NoticiasScreen` es un `StatelessWidget` que se reconstruye reactivamente gracias a `Provider.of<NoticiasProvider>`.

### Estructura visual

La pantalla se divide en tres estados bien diferenciados:

- **Cargando**: muestra un `CircularProgressIndicator` mientras `provider.cargando` es `true`.
- **Error**: muestra un icono de sin conexión, el texto de error localizado y un botón de reintento que llama a `provider.cargarNoticias()`.
- **Contenido**: lista las noticias separadas en dos secciones mediante un `ListView`.

### Secciones de lectura

Las noticias se dividen en dos grupos: las no leídas (con el contador de pendientes como cabecera) y las leídas. Cada sección tiene su propio encabezado visual generado por `_cabeceraSeccion`, que muestra una barra de color y el texto en mayúsculas.

### Tarjeta de noticia

Cada noticia se renderiza con el widget privado `_tarjetaNoticia`, que muestra:

- Una miniatura cuadrada (imagen de red si existe, o el icono de campaña con color de fondo diferenciado según estado de lectura).
- La fecha relativa (calculada por el método `_formatearFecha`, que devuelve "Ahora", "Hace X min", "Hace X h", "Ayer", etc.).
- Título y subtítulo con el estilo tipográfico diferenciado entre leída (peso medio) y no leída (negrita).
- Un punto verde indicador para las no leídas, o un icono de check para las leídas.

Al pulsar la tarjeta se navega a `/noticia/{id}` con `context.push`.

---

## 6. Pantalla de detalle – `detalle_noticia_screen.dart`

`DetalleNoticiaScreen` recibe el `noticiaId` como parámetro de construcción y lo usa para recuperar la noticia del provider mediante `obtenerNoticiaPorId`.

### Marcado automático como leída

En `initState`, usando `addPostFrameCallback`, se llama a `provider.marcarComoLeida(noticiaId)` en el primer frame tras construirse el widget, garantizando que la noticia quede marcada como leída en el instante en que el usuario la abre.

### Layout

La pantalla usa `extendBodyBehindAppBar: true` para que la imagen ocupe el área de la barra superior, logrando un efecto inmersivo. La barra es transparente y solo contiene el botón de retroceso con un `CircleAvatar` semitransparente sobre la imagen.

El cuerpo es un `SingleChildScrollView` con dos bloques:

- **Cabecera de imagen**: 260 dp de alto. Si la noticia tiene `imagenUrl`, carga la imagen con `Image.network` y un `errorBuilder` de respaldo; si no, muestra un gradiente verde con el icono de campaña.
- **Tarjeta de contenido**: se superpone a la imagen con un `Transform.translate` de –28 dp (el radio del borde redondeado superior). Dentro se muestran la píldora de fecha, el título, el divisor decorativo verde, el subtítulo, un separador horizontal y el cuerpo completo de la noticia.

Si el ID no corresponde a ninguna noticia conocida, se muestra un `Scaffold` mínimo con el mensaje `news_detail_notFound`.

---

## 7. Internacionalización – Traducciones

La aplicación soporta dos idiomas: español (`es`) e inglés (`en`), implementados mediante el sistema oficial de Flutter (`flutter_localizations` + `intl`).

### Estructura del sistema

- Los archivos fuente de traducción son `en.arb` y `es.arb`, en formato ARB (Application Resource Bundle). Cada clave tiene su valor en el idioma correspondiente.
- A partir de estos archivos, la herramienta `flutter gen-l10n` genera automáticamente tres clases: la clase abstracta `AppLocalizations` con todos los getters declarados, y las implementaciones concretas `AppLocalizationsEn` y `AppLocalizationsEs`.
- El delegate `_AppLocalizationsDelegate` resuelve en tiempo de ejecución qué implementación instanciar según el `Locale` activo.

### Claves del módulo de noticias

Me encargué de definir todas las cadenas relativas al módulo, entre ellas:

| Clave | ES | EN |
|---|---|---|
| `news_title` | Noticias | News |
| `news_markAllRead` | Todo leído | Mark all read |
| `news_loadError` | No se pudieron cargar las noticias | Could not load the news |
| `news_retry` | Reintentar | Retry |
| `news_empty` | No hay noticias disponibles | No news available |
| `news_unread(count)` | `{count} sin leer` | `{count} unread` |
| `news_sectionRead` | Leídas | Read |
| `news_detail_title` | Noticia | News |
| `news_detail_notFound` | Noticia no encontrada | News not found |
| `news_detail_share` | Compartir noticia | Share news |

La clave `news_unread` es un ejemplo de cadena parametrizada: recibe el entero `count` y lo interpola en la cadena, lo que en Flutter se declara como método en lugar de getter para poder pasar el argumento.

Además de las claves del módulo de noticias, participé en la definición de las claves comunes de navegación y ajustes que se usan en el resto de la aplicación.


---

Hay que decir que algunas traducciones no estan implementadas aquí pero en la aplicación completa si, ya que la he tenido que adaptar bastante y es nada más que para hacer una idea

