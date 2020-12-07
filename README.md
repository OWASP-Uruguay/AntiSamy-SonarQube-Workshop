# OWASP AntiSamy y SonarQube Workshop

Repository for [OWASP Uruguay](https://owasp.org/uruguay) Meetup: [Workshop OWASP AntiSamy y SonarQube 101](https://www.meetup.com/es/OWASP-Uruguay-Chapter/events/274852316/)

Repositorio para el Meetup organizado por OWASP Uruguay: [Workshop OWASP AntiSamy y SonarQube 101](https://www.meetup.com/es/OWASP-Uruguay-Chapter/events/274852316/)

## Introducción
En este workshop aprenderemos acerca del proyecto de OWASP AntiSamy y sobre la herramienta de SAST SonarQube

Video [aquí]()

## OWASP AntiSamy

Presentación AntiSamy [aquí](https://docs.google.com/presentation/d/1TpP3BhdII8vTYWpnZk8ys4QMtlKh0q_6HS5EseJGR1M/edit?usp=sharing)

### Inicio de ambiente local

Se utilizará un fork del proyecto [simplecommerce/SimplCommerce](https://github.com/spassarop/SimplCommerce) como proyecto de ejemplo a modificar.

Los requisitos del paso a paso para el ambiente de demo pueden variar, aquí se describe uno de ellos:

1.  Descargar e instalar [NodeJS](https://nodejs.org/en/download/).
2.  Descargar e instalar [Visual Studio 2019](https://visualstudio.microsoft.com/vs/community/) con el SDK y runtime de [.NET 5.0](https://dotnet.microsoft.com/download/dotnet/5.0). En caso de seleccionar componentes indviduales en la instalación, asegurarse de marcar *".NET Core cross-platform development"*, esto debería instalar lo necesario para usar .NET 5.0.
3.  Descargar e instalar [SQL Server](https://www.microsoft.com/en-us/sql-server/sql-server-downloads). La forma más sencilla y descartable es utilizar Docker, siguiendo la [guía en contenedores Linux](https://docs.microsoft.com/en-us/sql/linux/quickstart-install-connect-docker?view=sql-server-ver15&pivots=cs1-powershell) (puede ser también con contenedores Windows):
    1.  Descargar la imagen oficial en su última versión:  
        ```powershell
        docker pull mcr.microsoft.com/mssql/server:2019-latest
        ```
    2.  Iniciar un contenedor estableciendo una contraseña para el usuario `SA`:  
        ```powershell
        docker run -e "ACCEPT_EULA=Y" -e "SA_PASSWORD=<YourStrong@Passw0rd>" `
        -p 1433:1433 --name sql-server -h sql-server `
        -d mcr.microsoft.com/mssql/server:2019-latest
        ```
    3.  Probar el servicio y conexión con el comando:
        ```powershell
        docker exec -it sql-server /opt/mssql-tools/bin/sqlcmd -S localhost -U SA -P "<YourStrong@Passw0rd>"
        ```
        Si se muestra la línea `1>`, el login fue exitoso. Se puede salir con el comando `exit`.
4.  Clonar el repositorio del proyecto:
    ```powershell
    git clone https://github.com/spassarop/SimplCommerce.git
    ```
5.  Abrir la solución y modificar el string de conexión en `SimplCommerce\src\SimplCommerce.WebHost\appsettings.json`:
    ```json
    "ConnectionStrings": {
        "DefaultConnection": "Server=.;Database=SimplCommerce;MultipleActiveResultSets=true;User Id=sa;Password=<YourStrong@Passw0rd>;"  
    }
    ```
6.  Compilar la solución.
7.  Establecer `SimplCommerce.WebHost` como Startup Project si no lo está.
8.  Abrir la ventana Package Manager Console. Allí ejecutar el comando `Update-Database`. Esto crea la base de datos a base de migraciones existentes en la solución.
9.  En Visual Studio presionar "Control + F5".
10. Entrar a la URL de la aplicación y crear datos para la industria de tipo "Fashion". El back-office del sitio es accesible vía `/Admin` con las el usuario `admin@simplcommerce.com` y contraseña `1qazZAQ!`.

### Ejecutando el ataque
1.  Con sesion iniciada como administrador, acceder a un nuevo producto mediante el cabezal de la web en Catalog > Products y luego con el botón "+ Create Product", o accediendo directamente en `/Admin#!/product-create`.
2.  Cargar datos arbitrarios para el producto. Particularmente en cualquiera de los campos que permiten ingresar "texto enriquecido" (HTML a fin de cuentas) presionar el botón `</>` (Code View) para ver el HTML resultante de la escritura.
3.  Allí insertar el siguiente HTML: 
    ```html
    <p>Una camiseta <b>con toda la onda</b>.</p><p><script>alert(document.cookie)</script>Unisex.</p>
    ```
4.  Guardar el producto, dirigirse al nuevo producto en la raíz del sitio web y acceder a la publicación.
5.  Ver la alerta al cargar la página, con las cookies accesibles por JavaScript.

### Mitigando

#### Implementar el código sanitizador
Aquí entra en juego [OWASP AntiSamy .NET](https://github.com/spassarop/antisamy-dotnet). La forma más sencilla de incluirlo es instalando su correspondiente [paquete NuGet](https://www.nuget.org/packages/OWASP.AntiSamy/). Esto puede hacerse con los siguientes pasos:
1.  Click derecho sobre el proyecto `SimplCommerce\src\Modules\SimplCommerce.Module.Catalog`.
2.  Click en "Manage NuGet packages...".
3.  Buscar "antisamy", seleccionar el paquete "OWASP.AntiSamy" y hacer click en instalar.

Una vez instalado, puede crearse una nueva clase en el mismo proyecto llamada `HtmlSanitizer.cs` con la siguiente estructura:
```csharp
using OWASP.AntiSamy.Html;

namespace SimplCommerce.Module.Catalog
{
    internal static class HtmlSanitizer
    {
        public static string Santize(string badHtml)
        {
        }
    }
}
```
Dentro de la función `Sanitize` el código más simple se construye con los siguientes pasos:
1.  Retornar si `badHtml` es nulo o vacío:
```csharp
if (string.IsNullOrEmpty(badHtml))
{
    return badHtml;
}
```
2.  Crear una instancia de AntiSamy:
```csharp
var antisamy = new AntiSamy();
```
3.  Ejecutar el escaneo con la política por defecto:
```csharp
CleanResults results = antisamy.Scan(badHtml, Policy.GetInstance());
```
4.  Retornar el HTML sanitizado:
```csharp
return results.GetCleanHtml();
```

#### Aplicar la sanitización

Para mitigar se deben cubrir dos frentes. La salida de la información (cuando se muestra en la vista) y la entrada (cuando es editado el producto). Como el texto malicioso ya fue ingresado, mitigar las salidas implica sanitizar el HTML en la carga del modelo y retorno de la API para el método GET:
-   `ProductController.cs` > `ProductOverview(long id)` y `ProductDetail(long id)`:
    ```csharp
    ShortDescription = _contentLocalizationService.GetLocalizedProperty(product, nameof(product.ShortDescription), HtmlSanitizer.Santize(product.ShortDescription)),
    ```
-   `ProductApiController.cs` > `Get(long id)`
    ```csharp
    ShortDescription = HtmlSanitizer.Santize(product.ShortDescription),
    ```
Con esto, para la propiedad `ShortDescription` (también debería hacerse para `Description` y `Specification`) se evita la ejecución de código JavaScript para el ejemplo de ataque utilizado. De hecho, el resultado pasa a ser:
```html
<p>Una camiseta <b>con toda la onda</b>.</p><p>Unisex.</p>
```

Para mitigar la inyección en futuras creaciones/modificaciones de producto, aplicar en:
-   `ProductApiController.cs` > `Post(ProductForm model)`:
    ```csharp
    ShortDescription = HtmlSanitizer.Santize(model.Product.ShortDescription),
    ```
-   `ProductApiController.cs` > `Put(long id, ProductForm model)`:
    ```csharp
    product.ShortDescription = HtmlSanitizer.Santize(model.Product.ShortDescription);
    ```
#### Mejoras en el proceso de depuración
Puede irse un paso más allá basarse en los resultados del escaneo para retornar un error al detectar entradas maliciosas. Para esto, se agrega en la clase `HtmlSanitizer` una nueva función:
```csharp
public static List<string> ValidateHtml(string badHtml)
{
    if (string.IsNullOrEmpty(badHtml))
    {
        return new List<string>();
    }

    var antisamy = new AntiSamy();
    CleanResults results = antisamy.Scan(badHtml, Policy.GetInstance());
    return results.GetErrorMessages();
}
```
Es similar a `Sanitize` pero retorna una lista con los errores detectados en lugar del HTML sanitizado. Luego puede aplicarse de la siguiente forma para retornar los errores al cliente y facilitar la comprensión del error:
-   `ProductApiController.cs` > `Put(long id, ProductForm model)` y `Post(ProductForm model)`:
    ```csharp
    var errors = HtmlSanitizer.ValidateHtml(model.Product.ShortDescription);
    if (errors.Count > 0)
    {
        return BadRequest(new { error = new string[] { "Errors in short description: " + string.Join(",\n", errors) } });
    }
    ```


## SonarQube

Presentación SonarQube 101 [aquí](https://drive.google.com/file/d/1z_ztxxj1O4oydbb2j_MvZcMuytbka-bO/view?usp=sharing)

Links de interes:
* SonarQube: https://www.sonarqube.org
* SonarQube Docker: https://hub.docker.com/_/sonarqube

Otros links:
* OWASP mantiene un listado de herramientas para SAST en:  https://owasp.org/www-community/Source_Code_Analysis_Tools
* SpotBugs: https://spotbugs.github.io
* PMD: https://pmd.github.io
* AppScan (HCL): https://www.hcltech.com/software/appscan-standard
* CxSAST: https://www.checkmarx.com/products/static application-security-testing/
* Fortify (Microfocus): https://www.microfocus.com/en/us/products/static-code-analysis-sast/overview
