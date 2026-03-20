# Historias de Usuario: MVP Red-Nómada

## Historia de Usuario 1: El "Snapshot" Técnico (Decisión en menos de 5 segundos)
**Como** profesional itinerante bajo presión de entregas,  
**quiero** visualizar un desglose técnico inmediato (*The Snapshot*) de un espacio,  
**para** decidir si es funcional para mis necesidades específicas sin perder tiempo leyendo reseñas subjetivas.

### Criterios de Aceptación:
*   **Prohibición de Notas:** La interfaz no debe mostrar ninguna puntuación general, ni estrellas ni puntajes del 1 al 10.
*   **Pilares Obligatorios:** Deben visualizarse exclusivamente los 3 iconos de respuesta rápida con sus valores fijos: **Conectividad** (Baja/Media/Alta), **Energía** (Pocos/Suficientes/Muchos) y **Ambiente** (Silencioso/Moderado/Ruidoso).
*   **UI Eficiente:** El diseño de la interfaz debe garantizar que el usuario pueda identificar el estado de estos 3 pilares y tomar una decisión en menos de 5 segundos.

---

## Historia de Usuario 2: El Agente de Verificación (Mitigación de Datos Obsoletos)
**Como** usuario activo de la comunidad que ya tiene la aplicación abierta en un espacio de trabajo,  
**quiero** recibir alertas visuales pasivas para verificar lugares con información próxima a expirar,  
**para** evitar que otros colegas lleguen a un sitio que ya no cumple con las condiciones reportadas debido a la desactualización de los datos.

### Criterios de Aceptación:
*   **Gestión de Permisos:** El sistema debe mostrar un aviso preguntando al usuario si desea tener los servicios de ubicación habilitados "siempre", "una vez" o "solo cuando está en uso la app", respetando su elección para no drenar su batería sin su consentimiento.
*   **Validación Pasiva en App (Sin Push obligatorias):** Basado en los permisos otorgados, el sistema debe solicitar la re-validación (ej. "¿Sigue tranquilo este café?") priorizando a los usuarios que estén utilizando la app en ese momento. Se prohíbe requerir el rastreo GPS en segundo plano o notificaciones push externas como única vía de funcionamiento .
*   **Etiqueta de Urgencia:** Los lugares con datos próximos a vencer (más de 2 horas) deben mostrar visualmente una alerta o banner dentro de la interfaz que indique: **"Este lugar necesita verificación AHORA"**.
*   **Input de un toque:** El usuario debe poder confirmar el estado actual del lugar desde la alerta de forma rápida para mantener el sistema vivo y actualizado en tiempo real.

---

## Historia de Usuario 3: Gamificación de Impacto Real
**Como** colaborador de la red nómada,  
**quiero** completar misiones de reporte y visualizar a cuántas personas he ayudado con mi actualización,  
**para** sentir que mi aporte tiene un valor tangible en la productividad de la comunidad y motivarme a seguir participando.

### Criterios de Aceptación:
*   **Misiones Activas:** El sistema debe proponer retos específicos para incentivar comportamientos útiles, como "Reportá 3 cafés hoy" para obtener insignias, o dar puntos adicionales por "Confirmar un lugar lleno" y un bonus por "Ser el primero en reportar este lugar".
*   **Métrica de Impacto Social:** Inmediatamente después de finalizar una re-validación o reporte, el sistema debe mostrar un mensaje de refuerzo indicando el impacto real, por ejemplo: **"Gracias a tu reporte, 12 personas evitaron ir a un lugar lleno"**.