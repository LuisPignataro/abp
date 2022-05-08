# Sistema de archivos virtual

El sistema de archivos virtuales permite administrar archivos que no existen físicamente en el sistema de archivos (disco).

## Instalación

> La mayoría de las veces no necesitará instalar manualmente este paquete, ya que viene preinstalado con la [plantilla de aplicacion](Startup-Templates/Application.md).

[Volo.Abp.VirtualFileSystem](https://www.nuget.org/packages/volo.abp.virtualfilesystem)) es el paquete principal del sistema de archivos virtual.

Use ABP CLI para agregar este paquete a su proyecto:

*   Instale el [ABP CLI](https://docs.abp.io/en/abp/latest/cli)), si no lo ha instalado.
*   Abra una línea de comandos (terminal) en el directorio del archivo `.csproj` que desea agregar el paquete`volo.abp.virtualfilesystem`.
*   Ejecute el comando `abp add-Package Volo.abp.VirtualFilesystem`.

Si desea hacerlo manualmente, instale el paquete nuget [Volo.abp.virtualFilesystem](https://www.nuget.org/packages/volo.abp.virtualfilesystem)) en su proyecto y agregue `[DependsOn(typeof(AbpVirtualFileSystemModule))]` a la clase [módulo ABP](Module-Development-Basics.md) dentro de su proyecto.

## Trabajar con los archivos incrustados

### Incrustar archivos

Un archivo debe marcarse primero como **recurso incrustado** para incrustar el archivo en el ensamblaje. La forma más facil de hacerlo es seleccionando el archivo en el **explorador de soluciones** y estableciendo **accion** a **recurso embebido** en la ventana de **propiedades**. Ejemplo:

![Build-Action-Embedded-Resource-Sample](Images/build-Action-Embedded-Resource-Sample.png)

Si desea agregar varios archivos, esto puede ser tedioso.
Alternativamente, puede editar directamente el archivo `.csproj`:

```C#
<ItemGroup>
  <EmbeddedResource Include="MyResources\**\*.*" />
  <Content Remove="MyResources\**\*.*" />
</ItemGroup>
```

Esta configuración agrega recursivamente todos los archivos en la carpeta **MyResources** del proyecto (incluidos los archivos que agregará en el futuro).

Incrustar un archivo en el proyecto/ensamblaje puede causar problemas si un nombre de archivo contiene algunos caracteres especiales.

1.  Agregue el paquete Nuget [Microsoft.extensions.fileproviders.embedded](https://www.nuget.org/packages/microsoft.extensions.fileproviders.embedded)) al proyecto que contiene los recursos incrustados.
2.  Agregue `<GenerateEmbedDedFilesManifest>true</GenerateEmbedDedFilesManifest>` en la sección `<PropertyGroup> ... </propertyGroup>` de su archivo `.csproj`.

> Si bien estos dos pasos son opcionales y ABP puede funcionar sin esta configuración, se sugiere encarecidamente que lo haga.

### Configurar las ABPVirtualFileSystemOptions

Use `ABPVirtualFilesystemOptions` [clase de opciones](Options.MD) para registrar los archivos incrustados en el sistema de archivos virtuales en el método `ConfigureServices` de su [módulo](Module-Development-Basics.md).

**Ejemplo: agregar archivos incrustados al sistema de archivos virtual**

```csharp
Configure<AbpVirtualFileSystemOptions>(options =>
{
    options.FileSets.AddEmbedded<MyModule>();
});
```

El método de extensión `AddEmbedded` toma una clase cualquiera, encuentra todos los archivos integrados del ensamblaje **de la clase dada** y los registra al sistema de archivos virtuales.

`Addembedded` puede obtener dos parámetros opcionales;

*   `Basenamespace`: esto solo puede ser necesario si no configuró el paso`GenerateEmbedDedFilesManifest` explicado anteriormente y su espacio de nombres raíz no está vacío.
*   `BaseFolder`: si no desea exponer todos los archivos integrados en el proyecto, y solo desea exponer una carpeta específica (y subcarpetas/archivos), puede configurar la carpeta base en relación con la carpeta raíz de su proyecto.

**Ejemplo: Agregue archivos en la carpeta `myresources` en el proyecto**

```csharp
Configure<AbpVirtualFileSystemOptions>(options =>
{
    options.FileSets.AddEmbedded<MyModule>(
        baseNamespace: "Acme.BookStore",
        baseFolder: "/MyResources"
    );
});
```

Este ejemplo asume;

*   Su espacio de nombres de Project Root (predeterminado) es `acme.bookstore`.
*   Su proyecto tiene una carpeta, llamada `MyResources`
*   Solo desea agregar la carpeta `MyResources` al sistema de archivos virtuales.

### IVirtualFileProvider

Después de incorporar un archivo en un ensamblaje y registrarlo en el sistema de archivos virtuales, la interfaz `IVirtualFileProvider` se puede usar para obtener el contenido del archivo o el directorio:

```C#
public class MyService : ITransientDependency
{
    private readonly IVirtualFileProvider _virtualFileProvider;

    public MyService(IVirtualFileProvider virtualFileProvider)
    {
        _virtualFileProvider = virtualFileProvider;
    }

    public void Test()
    {
        //Getting a single file
        var file = _virtualFileProvider
            .GetFileInfo("/MyResources/js/test.js");

        var fileContent = file.ReadAsString();

        //Getting all files/directories under a directory
        var directoryContents = _virtualFileProvider
            .GetDirectoryContents("/MyResources/js");
    }
}
```

## Integración con ASP.NET Core

El sistema de archivos virtuales está bien integrado a ASP.NET Core:

*   Los archivos virtuales se pueden usar como archivos físicos (estáticos) en una aplicación web.
*   JS, CSS, archivos de imagen y todos los demás tipos de contenido web se pueden incrustar en ensambles y usarse como los archivos físicos.
*   Una aplicación (u otro módulo) puede **sobre escribir un archivo virtual** de un módulo al igual que colocar un archivo con el mismo nombre y extensión en la misma carpeta del archivo virtual.

### Carpetas de archivos virtuales estáticos

Por defecto, ASP.NET Core solo permite que la carpeta `wwwroot` contenga los archivos estáticos consumidos por los clientes. Cuando se utiliza Virtual File System, las siguientes carpetas pueden contenter contenido estático:

* Pages
* Views
* Components
* Themes

Esto facilita agregar y mantener archivos `js`, `.css` ... dentro de su proyecto.

## Tratar con archivos incrustados durante el desarrollo

Incrustar un archivo en un ensamblaje y poder usarlo desde otro proyecto simplemente haciendo referencia al ensamblaje (o agregando un paquete NUGET) es invaluable para crear un módulo reutilizable.

Supongamos que está desarrollando un módulo que contiene un archivo JavaScript integrado.

Lo que se necesita es la capacidad de la aplicación para usar directamente el archivo físico en el tiempo de desarrollo para que una actualización del navegador refleje cualquier cambio realizado en el archivo JavaScript.

El siguiente ejemplo muestra una aplicación que depende de un módulo (`mymodule`) que contiene archivos integrados.

```C#
[DependsOn(typeof(MyModule))]
public class MyWebAppModule : AbpModule
{
    public override void ConfigureServices(ServiceConfigurationContext context)
    {
        var hostingEnvironment = context.Services.GetHostingEnvironment();

        if (hostingEnvironment.IsDevelopment()) //only for development time
        {
            Configure<AbpVirtualFileSystemOptions>(options =>
            {
                options.FileSets.ReplaceEmbeddedByPhysical<MyModule>(
                    Path.Combine(
                        hostingEnvironment.ContentRootPath,
                        string.Format(
                            "..{0}MyModuleProject",
                            Path.DirectorySeparatorChar
                        )
                    )
                );
            });
        }
    }
}
```

El código anterior supone que `myWebAppModule` y`mymodule` son dos proyectos diferentes en una solución de Visual Studio y `myWebAppModule` depende de `mymodule`.

> La [plantilla de aplicacion](Startup-Templates/Application.md) ya usa esta técnica para los archivos de localización.

## Reemplazar/sobre escribir archivos virtuales

El sistema de archivos virtuales crea un sistema de archivos unificado en tiempo de ejecución, donde los archivos reales se distribuyen en diferentes módulos en el tiempo de desarrollo.

Si dos módulos agregan un archivo a la misma ruta virtual (como `my-path/my-file.css`), el agregado luego anula/reemplaza el anterior [dependencia del módulo](Module-Development-Basics.md)

Esta característica permite que su aplicación anule/reemplace cualquier archivo virtual definido por un módulo que su aplicación utilice.

Entonces, si necesita reemplazar un archivo de un módulo, simplemente cree el archivo en la misma ruta exactamente en su módulo/aplicación

### Archivos físicos

Los archivos físicos siempre anulan los archivos virtuales. Esto quiere decir que si usted coloca un archivo en `/wwwroot/my-folder/my-file.css`, este va a sobre escribir el archivo en la misma ubicación del virtual file system.
Por lo tanto, debe conocer las rutas de archivo definidas en los módulos para anularlas.