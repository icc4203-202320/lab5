**Nombres de integrantes del grupo:** (completar aquí, y ver informaciones adicionales solicitadas al final de este documento)

# Proyecto 1.5 - Intermodalidad, sensibilidad al contexto y tiempo real

El contexto del usuario móvil es un constructo complejo que abarca múltiples variables: tiempo, ubicación geográfica, la tarea que el usuario realiza y el estado de la tarea, la interacción social (mediada por el software, y/o cara a cara), etc. Hemos visto en el curso cómo el espacio físico puede ser mejorado con ciertos dispositivos (_beacons_ BLE) o incluso medios analógicos (p.ej., papel impreso con códigos QR) que proveen información contextual a la aplicación móvil, en forma automática y transparente (_beacons_) o requiriendo al usuario una interacción mínima (NFC, códigos QR).

En esta última entrega de proyecto, el objetivo es completar ciertas funciones de la aplicación Travel Log que se relacionan con lo anterior:

1. [2.0; escala 1-10] Que un usuario pueda agregar a otro como amigo/compañero de viaje, y pueda quedar registrado el momento y el lugar en donde ocurre esta acción. Esta acción se podrá realizar generando y escaneando un código QR, similar a cómo lo hacen aplicaciones como WhatsApp.
2. [1.0; escala 1-5] Que el usuario pueda contar con un perfil y agregar en éste su foto de perfil (avatar).
3. [2.0; escala 1-5] Que el usuario pueda agregar a un post fotografías [.5], que las fotografías aparezcan desplegadas en el post [.5] en formato de galería adecuado para pantalla de dispositivo móvil [.5], y que se puedan ver individualmente, ampliadas [.5].
4. [1.0; escala 1-3] Además, la aplicación deberá ser puesta en operación en una plataforma o nube pública, sobre todo para hacer funcionar el uso de códigos QR.

A continuación entregamos ciertas indiaciones y recomendaciones sobre cómo desarrollar lo anterior.

## Compañeros de Viaje

Cada usuario de Travel Log puede tener compañeros de viaje y agregarlos mediante el proceso en la siguente figura. Suponiendo que hay dos usuarios, 1 y 2, el Usuario 1 genera en Travel Log un código QR para amistad que luego muestra al Usuario 2. El Usuario 2 escanea el código (que tiene vigencia de 60 segundos) y luego de esto Travel Log automáticamente crea la relación de amistad entre los dos usuarios. Si el código expira, el Usuario 1 puede volver a generarlo y el proceso se repite. El Usuario 2 ve un mensaje de error si escanea un código expirado. 

El escaneo del código se hace con la aplicación de cámara del teléfono (o aplicación de escaner de códigos QR), es decir, no se pide implementar la funcionalidad de escaneo en la aplicación de _frontend_ en React. El código QR debe contener la URL del _frontend_ de Travel Log, con un código aleatorio en una variable del _query string_ (p.ej, `fndtk` - _friendship token_). La aplicación React recupera el _friendship token_ y luego lo envía al _backend_ para que procese la solicitud de amistad. La llamada al _backend_ debe contener `fndtk`, y la ubicación actual (de acuerdo a lo visto en el proyecto 1.4) del usuario. 

<img src="imágenes/secuencia-amistad.png" alt="drawing" width="700"/>

### Modelo de amistad en ActiveRecord

En el _backend_ de la aplicación se requiere crear modelo `Friendship` que permita vincular usuarios distintos en relación de amistad. La tabla subyacente `friendship` es básicamente una tabla de unión con claves foráneas a dos usuarios. Se deben además implementar validaciones para evitar la duplicidad de relaciones de amistad, y las auto-referencias (un usuario no es amigo de sí mismo).

Migración:

```ruby
class CreateFriendships < ActiveRecord::Migration[7.0]
  def change
    create_table :friendships do |t|
      t.references :user, foreign_key: true, null: false
      t.references :friend, references: :users, null: false

      # Guardar coordenadas en donde se origina la amistad en JSON
      # Por detault, guarda {}
      t.string :gps_coordinates, default: "{}", null: false

      t.timestamps
    end

    # Agregar clave foranea friend_id, en friendships, a users
    add_foreign_key :friendships, :users, column: :friend_id

  end
end
```

Modelos:

```ruby
class Friendship < ApplicationRecord
  belongs_to :user
  belongs_to :friend, class_name: 'User'

  # Validaciones de unicidad y de no-autoreferencia
  validate :check_uniqueness
  validate :avoid_self_referencing

  private

  def check_uniqueness
    # Evita la duplicación (A, B) y (B, A)
    if Friendship.where(user_id: friend_id, friend_id: user_id).exists? ||
       Friendship.where(user_id: user_id, friend_id: friend_id).exists?
      errors.add(:base, "La amistad ya existe.")
    end
  end

  def avoid_self_referencing
    # Evita que un usuario sea amigo de sí mismo
    if user_id == friend_id
      errors.add(:base, "No puedes ser amigo de ti mismo.")
    end
  end
end

# Asociaciones para amistad en User
class User < ApplicationRecord
  has_many :friendships
  has_many :friends, through: :friendships
end
```

### Recuperación de variables de query strings en aplicación React

El componente `react-router-dom` provee un hook llamado `useLocation`. A través de este hook se puede acceder a las variables de query string en la URL actual. Además, se puede adjuntar un hook de efecto que actúe cada vez que la URL (y las variables de query string) cambian. El siguiente código ilustra cómo se implementa este comportamiento:

```javascript
import { useLocation } from "react-router-dom";

function FriendshipTokenHandler() {
    const location = useLocation();

    useEffect(() => {
        const queryParams = new URLSearchParams(location.search);
        const friendshipToken = queryParams.get('friendshipToken');
        
        if (friendshipToken) {
            // Enviar el token al backend para crear relación de amistad
        }
    }, [location.search]);
}
```

### Generación de código QR

El código QR con información de _friendship token_ deberá ser generado por el backend de Travel Log. Puedes definir `FriendshipToken` como un recurso singleton (es decir, sin `id`) en `routes.rb`. Luego, necesitas crear un controlador `FriendshipTokensController` que a través de la acción `show` genere un código QR de amistad para el usuario que llama al _endpoint_. 

Para generar el código QR, puedes usar la gema llamado [rqrcode](https://github.com/whomwah/rqrcode) (encuentra en la documentación el método `as_png`). Además, necesitarás incluir la gema llamada `chunky_png` para generar la imagen PNG. 

La acción `FriendshipTokensController::show` tendría que llamar a `as_png` de la siguiente manera:

```ruby
    send_data RQRCode::QRCode.new(url_de_frontend_con_fndtk_en_query_string).as_png(size: 300), type: 'image/png', disposition: 'attachment'
```

Con lo anterior, la aplicación de frontend puede descargar la imagen PNG con el código QR y desplegarla para que el Usuario 2 la escanee.

## Toma de fotografía de avatar

La toma de fotografía de avatar se puede realizar en la aplicación de _frontend_ React de manera simple invocando la aplicación de cámara externa desde un elemento de formulario de tipo `input`:

```html
<input type="file" accept="capture=camera,image/*">
```

Sin embargo, se puede dar una mejor experiencia al usuario sin requerir un cambio de aplicación si se opta por acceder a la cámara desde el propio navegador web. Existen para React numerosos ejemplos sobre cómo hace una captura de imagen en el navegador web desde una aplicación React - [aquí les dejo uno completo](https://codesandbox.io/s/react-camera-api-image-capture-forked-qxp3yy). La API web estándar subyacente es la de [navigator.mediaDevices](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/mediaDevices).

## Subidas de archivos de imagen a la aplicación de backend de Travel Log

Las subidas de fotografías al backend de la aplicación TravelLog se deben realizar utilizando [Active Storage](https://guides.rubyonrails.org/active_storage_overview.html) de Rails. La aplicación de _backend_ de Travel Log ya incorpora un modelo de `Media` (con archivo adjunto), y `Photo` (derivado de `Media`). Pueden realizar cambios a este diseño si lo estiman necesario.

Para subir archivos desde el _frontend_ en React, pueden ver buenos ejemplos, usando Axios, en [esta guía](https://www.filestack.com/fileschool/react/react-file-upload/).

## Despliegue de aplicación en sitios públicos

La aplicación Travel Log que hemos desarrollado se compone por un _frontend_ hecho de código estático HTML, ES6+ & JSX, y CSS, y un _backend_ transaccional en RoR. La aplicación de _frontend_ al ser código estático, puede alojarse en una variedad de alternativas. A continuación te sugerimos algunas:

* GitHub Pages: Tu aplicación de frontend puede ser publicada desde el mismo repositorio en donde se encuentra en GitHub. Además, cada vez que haces `push` al repositorio, la aplicación se actualiza automáticamente en GitHub Pages. Puedes ver [documentación sobre esto aquí](https://www.geeksforgeeks.org/deployment-of-react-application-using-github-pages/) (revisa del paso 2 en adelante). Puedes ver la [documentación oficial](https://docs.github.com/en/pages/getting-started-with-github-pages/about-github-pages) y en particular, las limitaciones (_Usage limits_).
* Netlify: Se integra con GitHub, de manera que puede publicar tu aplicación React en cuanto haces `push` al repositorio. Es más flexible que GitHub pages, pues permite usar varios dominios y permite mayor escalabilidad, entre otras ventajas. Puedes ver una [guía aquí](https://www.geeksforgeeks.org/how-to-deploy-react-app-on-netlify-using-github/) (puedes ver desde el paso 2).
* Firebase Hosting: Competidor de Netlify, se usa en forma muy similar. Ver [guía aquí](https://www.knowledgehut.com/blog/web-development/deploying-react-app-to-firebase#what-is-firebase-hosting?-%C2%A0). 

Para la aplicación de _backend_ necesitas utilizar una plataforma que permita la operación de la aplicación RoR. Esto, a modo de simplificar el despliegue, pues existen muchas alternativas, como la de usar un _Virtual Private Server_ y configurarle todo su software, pero ello no te lo recomendamos por todo el tiempo y esfuerzo de mantenimiento que requiere. Otras alternativas, como usar Kubernetes en una nube pública (Google Cloud o AWS EKS), también requieren un esfuerzo de configuración considerable. Para simplificar las cosas y lograr la mayor productividad posible, te recomendamos usar las siguiente dos alternativas:

* Render: Existe desde 2019 y al igual que Heroku (ver a continuación), permite la publicación muy simplificada de aplicaciones RoR. Mantiene una capa gratuita (_free tier_), por lo que para efectos de este proyecto la recomendamos como primera alternativa. Puedes ver [una guía](https://render.com/docs/deploy-rails) sobre cómo desplegar una aplicación Rails en Render.
* Heroku: La primera PaaS robusta para desplegar aplicaciones RoR, desde 2007. Heroku simplificó drásticamente el proceso de despliegue de aplicaciones Rails con su modelo de `git push heroku master`. Desde Noviembre de 2022, Heroku ya no ofrece una capa gratuita (_free tier_), sin embargo, el precio de la primera capa es muy accesible. Puedes ver [una guía](https://devcenter.heroku.com/articles/getting-started-with-rails7) sobre cómo desplegar una aplicación Rails en Heroku.

Es importante notar que tu aplicación React deberá ser configurada para contener la URL del _backend_ y realizar todas las llamadas a API _endpoints_ con base a dicha URL. Además, la aplicación Rails en el backend requerirá conocer la URL de la aplicación de _frontend_, pues al generar códigos QR necesitará esta información. Es conveniente que la aplicación Rails obtenga la URL del _frontend_ a través de una [variable de entorno](https://rubyhero.dev/environment-variables). Se usa el objeto `ENV` para esto, p.ej., `ENV.fetch('API_KEY')`.

### Configuración de CORS en Rails

[CORS (Cross-Origin Resource Sharing)](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) es una medida de seguridad implementada por navegadores web para restringir las solicitudes web a una página de un dominio diferente al dominio de la página que la lanzó. Con tus aplicaciones éste será el caso, pues los dominios de _frontend_ y _backend_ serán distintos (al menos que hagas que tu aplicación Rails contenga la aplicación React y la sirva desde el directorio _public_, pero no te recomendamos esta alternativa, debido a que esto consumirá capacidad de Puma, el servidor de aplicación en RoR, para atender solicitudes a _endpoints_ de API).

Debido a lo anterior, es necesario configurar la aplicación RoR del _backend_, para que el navegador web pueda realizar peticiones a ella desde la aplicación React. Para realizar esto los pasos en general son los siguientes; en nuestro proyecto, **el primero (agregar la gema) ya está realizado**, pero el segundo lo puedes mejorar para aumentar la seguridad:

1. Agregar la gema `rack-cors`. En tu Gemfile, añade: 
```ruby
gem 'rack-cors'
```
2. Configurar CORS. En `config/application.rb` o en un archivo initializer (p. ej., `config/initializers/cors.rb` - este es el caso en la presente aplicación), añade (corrige):
```ruby
config.middleware.insert_before 0, Rack::Cors do
  allow do
    if Rails.env.development?
      # El puerto de localhost es aquel a través del cual la aplicación React es servida.
      origins 'localhost:3000', '127.0.0.1:3000'
    else
      # Reemplaza con el dominio correcto de tu app en GitHub Pages, Netlify u otra plataforma que hayas escogido. El dominio debiera venir desde una variable de entorno, recuperable con ENV.fetch
      origins 'tu-dominio-de-github-pages.com'
    end

    resource '*',
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options, :head]
  end
end
```

Aquí, `origins` especifica desde qué dominios se permitirán las solicitudes. Puedes especificar múltiples dominios separándolos por comas o usar un carácter * para permitir cualquier dominio (aunque esto último no es recomendado en producción por razones de seguridad).

## Forma de Trabajo

En esta entrega del proyecto deberán continuar su trabajo en el repositorio del Proyecto 1.3 (frontend). Además, podrán realizar modificaciones al backend (Proyecto 1.1) para actualizar controladores o incluso el modelo de datos (crear migraciones). Lo importante es que todo endpoint expuesto por el backend se mantenga REST. Se les revisarán sus commits a dicho repositorio. Se evaluará el último commit realizado hasta el 25 de septiembre a las 23:59 hrs.

## Informar aquí URLs, problemas conocidos y limitaciones

A continuación para la evaluación deben informar la URLs de sus aplicaciones de _frontend_ y _backend_:

* URL del _frontend_:
* URL del _backend_:

---

**Informar aquí problemas conocidos:**


---

**Informar aquí limitaciones en su aplicación:**

