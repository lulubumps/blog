---
title: "YouTube a Streamit: de una playlist a un catálogo de episodios en WordPress"
date: 2026-11-16 00:00:00 -0300
permalink: /youtube-a-streamit/
excerpt: "Cómo automaticé la carga de episodios de Esperanza TV, qué hace el plugin con cada video y cómo probar tu primera sincronización."
categories: [Proyectos, WordPress]
tags: [YouTube, Streamit, PHP, Automatización]
toc: true
toc_label: "En este artículo"
---

Cada vez que salía un video nuevo en YouTube, había que volver a hacer lo mismo para publicarlo en Esperanza TV. Abrir el video, copiar el título y la descripción, descargar la miniatura, crear el episodio y dejarlo en el programa y la temporada correctos. El trabajo era sencillo cuando se trataba de un video. Se volvía repetitivo cuando había que hacerlo para varios programas.

En [LinkedIn anuncié la versión 3.9.0](https://www.linkedin.com/feed/update/urn:li:activity:7525730571860205568/) y conté cómo esa tarea dio origen a **YouTube a Streamit**. Aquí quiero dejar por escrito lo que quedó fuera de ese anuncio. Qué lee el plugin, cómo decide si un episodio ya existe y qué pasa entre seleccionar un video y verlo en el catálogo de WordPress.

Este artículo describe la [versión 3.9.0](https://github.com/lulubumps/yt-to-streamit/releases/tag/yt-to-streamit-v3.9.0). El plugin es un proyecto independiente. Streamit es un producto comercial de Iqonic y se instala por separado.

## Una playlist alimenta un programa

En el panel vinculas un programa que ya existe en Streamit con una playlist de YouTube. La playlist es la fuente de videos. El programa es el destino, y dentro de él puedes repartir los episodios en varias temporadas.

| En YouTube | En WordPress y Streamit |
| --- | --- |
| Playlist | Fuente asociada a un programa existente |
| Video | Episodio con título, descripción, fecha y enlace o reproductor |
| Identificador del video | Referencia para reconocerlo aunque cambie el título |
| Miniatura | Imagen en la biblioteca de medios de WordPress |
| Duración, cuando está disponible vía API | Campo de duración del episodio |

El archivo de video permanece en YouTube. El plugin guarda una URL o un reproductor incrustado, según la configuración, y descarga la miniatura a WordPress. Si el video deja de estar disponible en YouTube, el episodio puede seguir en el catálogo, pero su reproducción queda sujeta a la disponibilidad del original.

## Cómo lee YouTube

Con una clave configurada, el plugin consulta **YouTube Data API v3**. Lee la playlist con `playlistItems`, en páginas de hasta 50 resultados, con un máximo de 10 páginas por lectura. Toma el ID del video, el título, la descripción y la fecha de publicación del video. Omite entradas privadas, eliminadas o sin una fecha de publicación utilizable. Luego consulta `videos` en grupos de hasta 50 IDs para obtener las duraciones.

El límite de lectura de hasta 500 entradas de playlist es distinto del límite de episodios creados por pasada. Si trabajas con una lista más grande, esta versión no recorre todo el historial. La fecha de corte también tiene una consecuencia práctica. Si agregas hoy a la playlist un video publicado hace seis meses, se evalúa con la fecha de hace seis meses.

Si falta la clave o falla la lectura de la playlist por API, se intenta el **feed RSS**, con hasta tres intentos. El feed ofrece una ventana más acotada de videos y el plugin no obtiene la duración por esa vía. Si tampoco puede leer el feed, deja el motivo en el registro para que se pueda distinguir un fallo de lectura de una omisión deliberada. [Código de lectura de YouTube](https://github.com/lulubumps/yt-to-streamit/blob/68054f14c1459a50b36bb8c26ee09941cbe6641e/includes/youtube.php)

## Cómo decide qué importar

El orden de estas comprobaciones importa. Un episodio que ya existe se omite incluso si el video está marcado para importar.

1. **Busca el ID de YouTube.** Primero comprueba si ya lo guardó como metadato de una importación anterior. También busca ese ID en los enlaces y reproductores incrustados de episodios existentes de Streamit. Así reconoce un episodio cargado a mano aunque su título sea distinto. Si el episodio antiguo usa un archivo local, no hay un ID de YouTube que comparar. En ese caso se puede excluir el video desde el panel.
2. **Respeta las decisiones explícitas.** Un video marcado para importar puede pasar aunque se haya publicado antes de la fecha de corte. Un video excluido se omite.
3. **Aplica el modo del programa.** En **Solo los que elija**, deja fuera todo video sin seleccionar. En **Todos los nuevos**, considera los que superan el filtro de fecha y las exclusiones.

Por ejemplo, si la playlist trae «Conversaciones | Capítulo 04» y Streamit ya tiene «Cómo nació el proyecto» con la misma URL de YouTube, el plugin reconoce que son el mismo video. El título distinto no crea un segundo episodio ni modifica el que ya estaba en el catálogo. [Código de selección y detección](https://github.com/lulubumps/yt-to-streamit/blob/68054f14c1459a50b36bb8c26ee09941cbe6641e/includes/videos.php)

## Qué crea en Streamit

La 3.9.0 permite hacer el trabajo editorial antes de importar. En **Videos de la playlist** puedes editar el título, elegir una temporada por video, aplicar una temporada a varios videos marcados, quitar un prefijo común de sus títulos o crear una temporada desde esa pantalla.

Supongamos que tienes tres videos marcados de «Conversaciones» y que la segunda temporada ya contiene cuatro episodios. Puedes elegir **Temporada 2** para los tres videos y quitarles el prefijo «Conversaciones |». Si el plugin puede importarlos todos, los procesa del más antiguo al más nuevo y les asigna los números 5, 6 y 7. Si un video no tiene temporada propia, se usa la configurada para el programa. Si tampoco hay una válida a ese nivel, se usa la última disponible. Cuando el programa aún no tiene temporadas, la importación crea la primera.

Para cada video elegible, la rutina de importación sigue este flujo:

1. **Crea el episodio con Streamit.** Usa `streamit_add_episode()` con el título elegido, la descripción de YouTube, el autor de WordPress y el estado configurado, por ejemplo borrador. La dirección del episodio deriva del título e incorpora el ID para distinguir videos con nombres iguales.
2. **Escribe los metadatos.** Guarda el programa, el número en la temporada, la fecha, el ID del video y el método de reproducción. Si hay un episodio modelo, puede heredar otros campos de ese modelo. Los datos propios del video, incluida la duración, se calculan por separado.
3. **Descarga la miniatura.** Prueba primero la versión de máxima resolución y luego la estándar de alta calidad. Si está habilitado WebP y el servidor puede convertirla, guarda la imagen en ese formato. Usa JPG si no puede convertirla. Si ambas descargas fallan, el episodio puede quedar sin miniatura.
4. **Enlaza el episodio con su temporada.** Agrega su ID a la lista de episodios, actualiza el metadato `_seasons` del programa e invalida la caché correspondiente. Ese enlace permite que el episodio aparezca dentro del programa, además de existir como registro.

Por ejemplo, un video de 1 hora, 5 minutos y 32 segundos queda con duración `1:06` en Streamit, porque el campo se guarda en horas y minutos redondeados. Si no se pudo obtener la duración, queda vacía. La herramienta **Completar duraciones** puede llenar esos campos más adelante sin sobrescribir los que ya tienen un valor. [Rutina de importación](https://github.com/lulubumps/yt-to-streamit/blob/68054f14c1459a50b36bb8c26ee09941cbe6641e/includes/importer.php)

## Una primera sincronización, paso a paso

Los requisitos declarados son **WordPress 5.8+**, **PHP 7.4+** y **Streamit 1.3.x** activo. La versión de Streamit con la que se probó esta integración es la 1.3.9.

1. Descarga el ZIP desde [GitHub Releases](https://github.com/lulubumps/yt-to-streamit/releases/tag/yt-to-streamit-v3.9.0). En WordPress, entra a **Plugins → Añadir nuevo → Subir plugin**, instala el ZIP y actívalo.
2. En **YouTube a Streamit → Ajustes**, elige URL o código incrustado según cómo funcione la reproducción en tu instalación. Configura la clave de YouTube Data API v3 para usar la API y obtener las duraciones. Puedes ingresarla en Ajustes o definir `YTS_API_KEY` en `wp-config.php`. La constante tiene prioridad.
3. En **Programas sincronizados**, elige el programa que ya existe en Streamit y pega el enlace o el ID de su playlist. Define una fecha de inicio y usa **Borrador** como estado inicial.
4. Para una prueba controlada, selecciona **Solo los que elija**. Abre **Videos de la playlist**, marca un video que aún no exista y elige su temporada. Corre **Verificación del sistema**. La opción **Verificar + prueba de creación** comprueba también si Streamit acepta crear un episodio de prueba.
5. Importa el video desde el panel y abre **Registro de sincronización**. Revisa que se creó un solo episodio y compara en Streamit el título, la temporada, la miniatura y la reproducción. Repite la sincronización y confirma que ese ID aparece como omitido por existir. Tras revisar el borrador, puedes publicarlo en Streamit y cambiar el estado de futuras importaciones según tu flujo editorial.

## Sincronización, avisos y mantenimiento

La frecuencia predeterminada es **cada 12 horas** y se puede cambiar a 30 minutos, 1 hora, 6 horas o 24 horas. El plugin usa **WP-Cron**. Con su configuración habitual, una tarea vencida se pone en marcha con una visita al sitio, así que un portal con poco tráfico puede ejecutarla después de la hora prevista.

Por defecto se crean hasta **10 episodios por programa por pasada**. Si quedan videos pendientes por ese límite, el plugin agenda una continuación para dos minutos después, sujeta también a WP-Cron. Un bloqueo evita que corran dos importaciones a la vez.

El aviso por correo usa el **total de episodios del programa**. Si configuras un aviso al llegar a 13, el plugin comprueba si el total llegó o superó esa cifra. Guarda que ya avisó cuando `wp_mail()` informa éxito. Si el envío falla, vuelve a intentarlo en otra pasada. La entrega final depende de la configuración de correo de WordPress.

| Si ves esto | Revisa esto primero |
| --- | --- |
| El video aparece como omitido | Abre el registro. Distingue entre episodio existente, exclusión manual y fecha de publicación anterior al corte. |
| El video queda pendiente | La pasada alcanzó el máximo de importaciones. Revisa la siguiente ejecución o sincroniza el programa manualmente. |
| El episodio existe pero no figura en el programa | Revisa la temporada y usa **Herramientas → Reparar enlaces** si quedó sin vincular. |
| Falta la duración | Comprueba que la API esté configurada y que el video tenga una duración disponible. Luego prueba **Completar duraciones**. |
| La playlist no carga | Mira el error en el registro. Comprueba el ID de la lista y si la API respondió con un problema de acceso o cuota. |

El **Registro de sincronización** muestra el resultado de la última ejecución guardada. **Episodios importados** permite consultar lo creado y **Verificación del sistema** ayuda a revisar la compatibilidad de la instalación. El panel se adapta a pantallas pequeñas y sigue el idioma de WordPress, con traducciones para variantes de español y portugués además del inglés base. [Guía de mantenimiento](https://github.com/lulubumps/yt-to-streamit/blob/68054f14c1459a50b36bb8c26ee09941cbe6641e/MANTENIMIENTO.md)

## Lo que me dejó el proyecto

La primera idea cabía en una línea. Traer videos de YouTube a WordPress. La parte que más me hizo aprender apareció cuando empecé a preguntarme qué debía pasar si el video ya existía, si el programa tenía varias temporadas o si el servidor alcanzaba su límite a mitad de una sincronización.

Durante el desarrollo usé IA para explorar opciones, revisar cambios y entender mejor cómo ordenar el código. Después venía el trabajo de decidir qué tenía sentido, probarlo y comprobar qué veía la persona que iba a usar el catálogo. Un episodio podía estar bien guardado y aun así faltar en la temporada. Esa diferencia fue la que me enseñó a verificar el recorrido completo.

Si vuelvo a este proyecto dentro de unos años, quiero poder reconstruir por qué un video terminó en una temporada, quedó pendiente o se omitió. Dejar esa decisión a la vista en el registro es parte del trabajo de automatizar.

[Código fuente de la 3.9.0](https://github.com/lulubumps/yt-to-streamit/tree/68054f14c1459a50b36bb8c26ee09941cbe6641e) · [Historial de versiones](https://github.com/lulubumps/yt-to-streamit/blob/68054f14c1459a50b36bb8c26ee09941cbe6641e/CHANGELOG.md)
