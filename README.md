This is a [Next.js](https://nextjs.org) project bootstrapped with [`create-next-app`](https://nextjs.org/docs/app/api-reference/cli/create-next-app).

## Getting Started

First, run the development server:

```bash
npm run dev
# or
yarn dev
# or
pnpm dev
# or
bun dev
```

Open [http://localhost:3000](http://localhost:3000) with your browser to see the result.

You can start editing the page by modifying `app/page.tsx`. The page auto-updates as you edit the file.

This project uses [`next/font`](https://nextjs.org/docs/app/building-your-application/optimizing/fonts) to automatically optimize and load [Geist](https://vercel.com/font), a new font family for Vercel.

## ZeroPintas
El sistema debe mostrar al perfecto en tiempo real el estado de un alumno, el sistema debe manejar estados dinamicos. 
Un alumno podra tener 3 estados principales Ausente En Plantel/Sin Clase En clase

## Flujo del Sistema - Diagrama de Secuencia

El sistema conecta 3 actores con el backend en Next.js (Server Actions/API) y Prisma + DB.

### Actores
- **Alumno:** Solo escanea su credencial.
- **Maestro:** Gestiona la asistencia de su clase.
- **Prefecto:** Monitorea discrepancias.
- **Next.js:** Orquestador / API.
- **Prisma + DB:** Persistencia.

### Flujos

#### 1. Registro de Entrada a Escuela (Amarillo - Pasos 1-6)

El alumno escanea su código de barras -> Next.js hace `Alumno.findUnique({ where: { codigoBarras } })` -> Si existe, crea `RegistroEntradaEscuela.create({ idAlumno, fechaHora: now() })` -> Devuelve feedback visual en pantalla (Verde / Sonido)

#### 2. Toma de Asistencia en Clase (Azul - Pasos 7-12)
El maestro selecciona su clase `(id_materia, id_grupo)` -> El backend consulta `Clase.findUnique() + Alumnos del grupo` y retorna la lista de inscritos. El maestro marca `PRESENTE/AUSENTE` y guarda -> Se ejecuta `AsistenciaClase.createMany()` -> Confirmación de guardado

#### 3. Dashboard de Prefectura - Detección de Fugas (Rosa - Pasos 13-16)
Es la lógica clave del sistema. El prefecto carga el Dashboard:
1.  Next.js consulta asistencias cruzadas.
2.  **Lógica de negocio:** 
    - Busca entradas a la escuela de hoy.
    - Filtra alumnos del grupo en horario de clase.
    - Detecta quiénes NO tienen `AsistenciaClase` registrada como PRESENTE.
3.  Retorna lista de alumnos en discrepancia y muestra alerta en rojo: **"Alumno X está en la escuela pero no en clase"**
```mermaid
sequenceDiagram
    participant A as Alumno
    participant M as Maestro
    participant P as Prefecto
    participant N as Next.js (Server Actions/API)
    participant DB as Prisma + DB

    rect rgb(255, 255, 204)
        Note over A, DB: 1. REGISTRO DE ENTRADA A ESCUELA
        A->>N: 1. Escanea credencial (Código de Barras)
        N->>DB: 2. Alumno.findUnique({ where: { codigoBarras } })
        DB-->>N: 3. Regresa datos del Alumno (id_alumno)
        N->>DB: 4. RegistroEntradaEscuela.create({ idAlumno, fechaHora: now() })
        DB-->>N: 5. Confirmación de registro
        N-->>A: 6. Feedback visual en pantalla (Verde / Sonido)
    end

    rect rgb(21, 190, 247)
        Note over M, DB: 2. TOMA DE ASISTENCIA EN CLASE
        M->>N: 7. Selecciona su Clase (id_materia, id_grupo)
        N->>DB: 8. Clase.findUnique() + Alumnos del grupo
        DB-->>N: 9. Lista de alumnos inscritos
        M->>N: 10. Marca estatus (PRESENTE/AUSENTE) y guarda
        N->>DB: 11. AsistenciaClase.createMany()
        DB-->>N: 12. Registros guardados
    end

    rect rgb(250, 77, 74)
        Note over P, DB: 3. DASHBOARD DE PREFECTURA
        P->>N: 13. Carga / Actualiza Dashboard de Prefectura
        N->>DB: 14. Consulta asistencias cruzadas
        Note over N, DB: 1. Busca entradas a la escuela de hoy.<br/>2. Filtra alumnos del grupo en horario de clase.<br/>3. Detecta quiénes NO tienen AsistenciaClase como PRESENTE.
        DB-->>N: 15. Retorna lista de alumnos en discrepancia
        N-->>P: 16. Muestra alerta en rojo: "Alumno X está en la escuela pero no en clase"
    end
```