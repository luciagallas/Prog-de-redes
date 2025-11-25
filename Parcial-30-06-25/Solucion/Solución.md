# Parcial 1 – Ejercicio 1: Docker

## A) ¿Qué problema soluciona Docker?

Docker soluciona el problema de que una aplicación funcione distinto en cada máquina (“en mi máquina anda”).

Lo hace usando **contenedores**, que incluyen la aplicación junto con todas sus dependencias y configuración. Esto permite:

- **Portabilidad**: la misma imagen se ejecuta igual en cualquier servidor con Docker.
- **Aislamiento**: cada aplicación corre en su propio contenedor sin interferir con otras.
- **Despliegues reproducibles**: el entorno es siempre el mismo (desarrollo, testing, producción).

En resumen, evita problemas de configuración y compatibilidad entre entornos y facilita el despliegue.

---

## B) Explicación de 3 líneas del Dockerfile

```dockerfile
1 | FROM mcr.microsoft.com/dotnet/sdk:8.0
2 | WORKDIR /app
3 | COPY *.csproj ./
4 | RUN dotnet restore
5 | COPY . .
6 | RUN dotnet publish -c Release -o ./publish
7 | EXPOSE 5000
8 | ENTRYPOINT ["dotnet", "./publish/Server.dll"]
```

### 1. `FROM mcr.microsoft.com/dotnet/sdk:8.0`

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0
```
- Define la imagen base.
- Usa una imagen oficial de Microsoft con el SDK de .NET 8.0 ya instalado.
- Es el “sistema operativo + .NET” sobre el que se construye nuestra imagen.

### 2. `WORKDIR /app`

```dockerfile
WORKDIR /app
```
- Define el directorio de trabajo dentro del contenedor.
- A partir de esta instrucción, todas las líneas siguientes (COPY, RUN, etc.) se ejecutan dentro de /app.
- Sirve para organizar la estructura interna del contenedor y evitar rutas largas.

### 3. `RUN dotnet publish -c Release -o ./publish`
```dockerfile
RUN dotnet publish -c Release -o ./publish
```
- Ejecuta un comando durante el build de la imagen.
- dotnet publish:
    - Compila la aplicación.
    - Resuelve dependencias.
- Genera la versión lista para ejecutar.
-c Release: usa la configuración optimizada para producción.
-o ./publish: guarda los binarios finales en la carpeta publish dentro del contenedor.